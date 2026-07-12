# PrintHub Studio target architecture

## Product boundary

The ecosystem is split into two independently deployable products.

### Thingdex

- Owns inventory, locations, item types, relations and scanner workflows.
- Runs without PrintHub and keeps label printing disabled by default.
- May use an optional PrintHub connector for automatic label jobs.
- Does not own templates, printers, ZPL, media or device status.

### PrintHub Studio

- Owns templates, typed template fields, sample data, preview and rendering.
- Owns the desktop designer and the mobile quick-print workflow.
- Owns printer registrations and print workflow status.
- Sends rendered ZPL to ZebraTamer or a legacy raw-9100 target.
- Runs without Thingdex and supports arbitrary manual/API-provided data.

`LabelGallery` is superseded by the Templates, Quick print and Printers views in
PrintHub Studio. Its repository remains available during migration but is not a
required runtime component.

## Runtime topology

```text
PrintHub Studio web
        |
        v
PrintHub API -------- template store
    |   |
    |   +------------ render/preview
    |
    +---------------- ZebraTamer REST API
                            |
                            +---- USB/character-device Zebra printer
```

ZebraTamer announces `_zpl-agent._tcp.local.` and `_zpl-printer._tcp.local.`.
PrintHub discovers these announcements, or uses explicit agent URLs from
`ZPLGRID_ZEBRA_TAMER_AGENTS` when multicast DNS is unavailable.

## Template contract

Template layout and template metadata stay separate:

- `template` contains the deterministic zplgrid layout.
- `variables` is the public input contract for all consumers.
- `sample_data` is used by the designer and previews.
- `preview_target` defines the intended media geometry.

A variable may contain:

```json
{
  "name": "title",
  "label": "Title",
  "type": "text",
  "mode": "required",
  "placeholder": "Cordless drill",
  "source_hint": "entity.display_name"
}
```

`type`, `label`, `placeholder` and `options` make the mobile form useful without
Thingdex. `source_hint` is advisory: integrations can bind a compatible value,
while standalone users simply enter it.

## Optional Thingdex connector

Thingdex uses a `label_profile` instead of exposing templates and printers in
every create form:

```json
{
  "item_type_id": "...",
  "template_id": "asset-label",
  "printer_id": "schildkrote",
  "auto_print": true,
  "bindings": {
    "title": "description",
    "identifier": "id",
    "detail": "location.path"
  }
}
```

ThingdexUI sends only inventory data in the normal workflow. Thingdex resolves
the optional profile and creates an idempotent, persistent PrintHub job after
the inventory transaction. A missing PrintHub never prevents Thingdex from
starting or saving inventory. Operators see job state and retry failed jobs in
PrintHub Studio.

## Migration phases

1. Deploy the new PrintHub API and Studio alongside LabelGallery.
2. Register ZebraTamer printers in PrintHub and verify status/job handoff.
3. Migrate existing templates without changing their v1 layout.
4. Move operators to the mobile Quick print view.
5. Remove LabelGallery from the default Compose topology.
6. Deploy Thingdex label profiles and the simplified operational forms.
7. Archive LabelGallery after saved bookmarks and draft links point to Studio.

## Release gates

- PrintHub starts and prints without Thingdex.
- Thingdex starts and saves inventory without PrintHub.
- Designer is usable at desktop widths; phones receive a clear Quick print path.
- Quick print works at 390 px without horizontal overflow.
- ZebraTamer jobs return a job id and state to PrintHub.
- Raw-9100 printers remain supported during migration.
