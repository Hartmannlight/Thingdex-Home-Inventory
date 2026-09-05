# PrintHub Studio target architecture

## Product boundary

The ecosystem is split into independently deployable products and bounded
runtime services. Product boundaries do not force unrelated failure domains
into one process.

### Thingdex

- Owns inventory, locations, item types, relations and scanner workflows.
- Runs without PrintHub and keeps label printing disabled by default.
- Uses a transactional outbox and a separately running PrintHub connector for
  automatic label jobs.
- Does not own templates, printers, ZPL, media or device status.

### PrintHub Studio

- Owns templates, typed template fields, sample data, preview and rendering.
- Owns the desktop designer and the mobile quick-print workflow.
- Owns logical print jobs and their preparation status.
- Submits immutable device artifacts to PrinterFleet.
- Runs without Thingdex and supports arbitrary manual/API-provided data.

### PrinterFleet

- Owns physical printer registrations, media observations, routing, delivery
  attempts, RAW TCP/serial-over-TCP and device status.
- Connects directly to reachable network printers.
- Uses PrintAgent only for USB, Bluetooth, local serial or isolated networks.
- Does not own templates, inventory data, scaling, dithering or preview.

### IPP gateway and PrintAgent

- The optional IPP gateway advertises a CUPS queue and forwards original source
  documents plus IPP tickets to PrintHub.
- PrintAgent is an optional edge process. ZebraTamer is its current compatible
  implementation; network printers do not require it.

`LabelGallery` is superseded by the Templates, Quick print and Printers views in
PrintHub Studio. Its repository remains available during migration but is not a
required runtime component.

## Runtime topology

```text
PrintHub Studio web
        |
        v
PrintHub API -------- template store
        |
        +------------ document preparation / preview
        |
        v
  PrinterFleet ------- direct RAW 9100 Zebra
        |
        +------------ PrintAgent ------- USB Zebra / future Niimbot
```

PrintAgent announces `_print-agent._tcp.local.`. PrinterFleet, not PrintHub or Thingdex, owns
discovery and explicit agent registration.

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
the optional profile and commits an immutable PrintIntent in the same database
transaction as the inventory object. A separate worker submits it idempotently
to PrintHub. A missing PrintHub never prevents Thingdex from starting or saving
inventory. Signed, replay-safe events return PrintHub state to Thingdex.

## Migration phases

1. Deploy the new PrintHub API and Studio alongside LabelGallery.
2. Register physical printers and PrintAgents in PrinterFleet and verify status
   and idempotent delivery handoff.
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
- Thingdex inventory and PrintIntent commit atomically while PrintHub is down.
- An expired worker lease resubmits with the same idempotency key.
- The integrated stack runs Thingdex API and print worker as separate processes.
