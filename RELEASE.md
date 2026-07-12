# Release process

Thingdex uses synchronized Semantic Versioning tags across the backend, both SDKs, and their consumers. A production release is made only from a source set for which every contract and consumer check passes.

## Prepare

1. Change and test the backend that owns the API contract.
2. Export and commit its OpenAPI document.
3. Copy the document into the corresponding SDK and regenerate the checked-in TypeScript schema.
4. Run the SDK CI command and update its changelog.
5. Update and build every UI consumer against those exact SDK sources.
6. Update component changelogs and bump compatible versions.

## Release

1. Merge the coordinated branches only after backend, SDK, and consumer CI is green.
2. Create the same `vMAJOR.MINOR.PATCH` tag in the backend, SDK, and UI repositories.
3. UI image workflows check out the SDK repositories at that matching tag, so a released image cannot silently use a newer contract.
4. Deploy immutable GHCR image tags; do not deploy `latest` directly.
5. Run database migrations, `/health` smoke checks, one read workflow, one create workflow, and a controlled label reprint.

## Contract compatibility

- Removing or renaming a path, field, enum value, or changing nullability requires a major version.
- Adding optional fields or endpoints is minor.
- Documentation and implementation-only fixes are patch releases.
- Generated files may never be edited by hand; CI rejects drift between OpenAPI input and generated SDK types.

SDK packages remain private workspace artifacts for now. Production images build them from matching tags. Registry publication can be added later without changing the contract process.
