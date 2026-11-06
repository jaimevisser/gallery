# External Library Owner Transfer and Metadata Portability

Date: 2026-11-05

## Status

Investigation and proposed direction. No implementation decision has been made.

## Problem

An external library is assigned to one Gallery user when it is created. The admin UI explicitly says that the owner cannot be changed later, and `UpdateLibraryDto` does not accept an owner. This is inconvenient when an administrator wants to move a library to another account, and deleting/recreating the library is not a safe workaround: assets and their database-only metadata are removed with the library.

The desired outcome is to change the account associated with a library while retaining as much metadata as possible, including people, tags, and locations. Two approaches have been suggested:

1. **Transfer the library in place** to another user and update the database consistently.
2. **Persist metadata beside the media** so removing and re-adding a library can restore it.

These approaches solve related but different problems. An in-place transfer can retain Gallery asset IDs and database relationships. Sidecars can make selected metadata survive asset deletion/recreation and be portable to other tools. Neither approach alone preserves every kind of metadata.

## Current behavior and findings

### A library's owner is not a single database field

- `library.ownerId` is non-null and cascades on user deletion. Library creation stores the selected owner. The update DTO and endpoint do not expose an owner change.
- Every external asset is imported with `asset.ownerId = library.ownerId`; the owner is copied when `LibraryService.processEntity` constructs an asset. Updating only `library.ownerId` would leave existing assets assigned to the old user.
- `asset.ownerId` is used for ownership, user-scoped queries, authorization, duplicate/checksum uniqueness, favorites, and face projections. `asset_face.ownerId` is also copied from the asset by database triggers and is used by face queries.
- The fork's `library_user` table records per-user library access grants for both the owner and shared-space paths. Its insert/delete triggers and the library audit tables drive sync backfills and access revocation. An owner transfer must reconcile these grants and sync events; changing the library row alone would leave stale or missing client data.
- A deleted library is soft-deleted first, then its assets are deleted from the database in background work. For external libraries the media files are not deleted, but their Gallery asset IDs and database-only associations are not retained for a later re-import.

### Which data can be retained or transferred

Keeping the same `library.id` and `asset.id` is the strongest preservation path. Rows keyed directly by an asset ID—such as EXIF metadata, edits, trims, and asset-to-face links—can remain attached without rescanning or rebuilding those IDs. This is materially safer than deleting and recreating assets.

However, several relationships carry user or scope semantics and need an explicit transfer policy:

| Data | Current scope / transfer consideration |
| --- | --- |
| Asset owner | `asset.ownerId` must move with `library.ownerId`; cached owner/visibility fields on faces must remain consistent. |
| People and faces | `person` rows are owner-scoped; face geometry is keyed to the asset, but face assignments point to person groups and Gallery identity side tables. A person may have faces on assets outside this library, so moving a whole person can affect unrelated assets. Identity/face records cannot be safely reassigned by changing just `person.ownerId`. |
| Tags | Gallery tags belong to a user (`tag.userId`); `tag_asset` links a tag to an asset. The target user may have a same-named tag or a different hierarchy, so tags need mapping/recreation and link remapping rather than a blind owner update. EXIF/XMP tag strings are a separate representation. |
| Favorites | Favorites are per-user in `asset_favorite`; `asset.isFavorite` is the owner's projection. The transfer needs a choice about whether to copy the old owner's favorite state to the new owner, leave it with the old user where access remains, or reset it. |
| Albums and stacks | Albums and stacks have owners and can include assets beyond the transferred library. Their membership may survive with the asset IDs but remain owned by the old account; moving an entire mixed album/stack would affect non-library assets. |
| Shared spaces | Links refer to the library ID, and space people are independently scoped to a space. Existing links may remain structurally valid, but membership grants, sync visibility, face matching, and cached client state need reconciliation after assets change owner. |
| Library sync and audit | `library_user`, library/asset audit rows, update IDs, and client backfills must cause the old owner's clients to remove access and the new owner's clients to receive the library and its assets. |

The current model therefore makes transfer **possible**, but not as a one-column update. A transfer has account-level consequences that must be treated as part of the operation, not as incidental cleanup.

### What file sidecars preserve today

Gallery already reads and writes a limited set of XMP metadata: description, rating, date/time, GPS coordinates, and tags. Writes merge those fields into an existing sidecar. A writable external-library mount is required for Gallery to persist changes there; a read-only mount cannot support this durability guarantee.

The metadata extraction service also imports named face regions from XMP `RegionInfo` (MWG-style regions): it uses the region name and rectangle to create Gallery face/person records. This is an **import** capability, not a save-back path. The current sidecar writer does not serialize face rectangles or person assignments. The existing face-region metadata work is useful precedent for parsing a standard format, but it does not by itself make Gallery edits portable.

Gallery's `person_identity` and `person_pet` tables are sometimes called person sidecars in code. They are database side tables used to keep fork-only fields out of upstream tables; they are not files beside the external media and do not survive deleting the asset/person rows.

XMP cannot naturally preserve all Gallery relationships. Album membership, stacks, user favorites, hidden-person state, birthdays, merge/identity decisions, shared-space memberships, and library configuration are not represented by the current XMP mapping. Encoding all of these would require a Gallery-specific, versioned metadata format and explicit conflict and privacy rules. Even then, read-only mounts and sidecar deletion/movement remain failure cases.

## Code and design references

- [Library DTO](../server/src/dtos/library.dto.ts), [library service](../server/src/services/library.service.ts), and [library table](../server/src/schema/tables/library.table.ts) define owner selection and current update/import behavior.
- [Asset table](../server/src/schema/tables/asset.table.ts), [asset-face table](../server/src/schema/tables/asset-face.table.ts), [tag table](../server/src/schema/tables/tag.table.ts), and [tag-asset table](../server/src/schema/tables/tag-asset.table.ts) show the owner and relationship scopes discussed above.
- [Library user table](../server/src/schema/tables/library-user.table.ts) and the [library access backfill design](2026-04-11-library-user-access-backfill-design.md) describe per-user library grants and sync backfills.
- [Metadata service](../server/src/services/metadata.service.ts) and [XMP sidecar documentation](../docs/docs/features/xmp-sidecars.md) describe current metadata writes and face-region imports.
- [Person sidecar helper](../server/src/gallery/utils/person-sidecar.ts) and [external-library face identity test design](2026-05-08-external-library-space-face-identity-tests-design.md) cover the distinct database-sidecar and shared-space face concerns.

## Option A: transfer the library in place

### Feasibility

This is the best path for retaining existing Gallery-only metadata because it keeps the library and asset identities stable. It also avoids touching external media paths or re-running file discovery. A dedicated admin operation is preferable to adding `ownerId` to the ordinary library-update DTO: this is a destructive-to-access ownership transition, not a routine setting edit.

### Required behavior

The operation should validate that the source library is active and the target account can own it, then preserve the library ID, asset IDs, import paths, and exclusions. Scans and ownership-sensitive writes must not race with the transfer. The service must update `library.ownerId`, update all associated `asset.ownerId` values, and reconcile trigger-maintained face state, `library_user`, audit records, update IDs, and sync backfills as one well-defined state transition.

User-scoped associations need explicit rules:

- Copy or map tags to the new user and retain the asset-tag links; do not silently point new-owner assets at tags belonging only to the old user.
- Decide whether the old owner's favorite state becomes the new owner's favorite state. Keep other viewers' favorites separate.
- Do not automatically transfer mixed albums, stacks, or people that also reference assets outside this library. Define whether those relationships stay with the old owner, are copied/split, or require the administrator to resolve them.
- Reconcile person/identity assignments per affected face. Preserve space-scoped people when valid, and queue the existing space/library face reconciliation for linked spaces. Avoid broad face re-detection unless the relationship repair cannot preserve valid assignments.
- Ensure the old owner's clients receive revocation/removal events and the new owner's clients receive a complete library/asset backfill. Sync state must not depend on either client being online during the operation.

For large libraries, a single transaction that updates every row may create unacceptable locks and WAL volume. The implementation should first define the concurrency and visibility contract, then choose between a bounded transaction and a resumable transfer job. A job-based design needs a durable transfer state, scan/write exclusion, idempotent batches, a clear rollback/retry path, and a finalization step so clients do not observe a partially transferred library as complete.

## Option B: persist metadata in sidecars

### Feasibility

This is feasible for metadata with a portable per-asset representation, but it is not a replacement for ownership transfer. The existing XMP writer provides a foundation for the currently supported scalar fields. Face-region import provides a standard-format starting point for persisting named rectangles.

### Additional design required

1. Define the portable fields and representation. Extend the existing MWG/XMP region support for face rectangles and names; do not assume that a person name alone is a stable cross-photo identity. If stable identity, birthday, pet species, or aliases must survive, specify a versioned identity key and a companion person/catalog representation.
2. Decide which source wins when Gallery edits and an external application edits the same sidecar. Preserve unrelated XMP data, handle deletions explicitly, and avoid re-importing stale sidecar values over newer manual Gallery decisions.
3. Keep provenance for imported versus manually assigned faces. Current metadata extraction replaces imported `SourceType.Exif` faces from the sidecar. A write-back implementation must ensure manual moves/names are represented and are not reverted by later extraction or ML jobs.
4. Make read-only behavior explicit. A sidecar-based preservation guarantee must fail visibly or clearly indicate that metadata is database-only when the mount cannot be written; silently accepting an edit that will later revert is not a preservation contract.
5. Separate portable per-file metadata from account/relationship data. Tags may be exported as strings but not their Gallery tag IDs or hierarchy. Albums, stacks, favorites, and shared-space relationships need different storage and ownership semantics.

Sidecars are particularly valuable for recovering supported metadata after a library is removed and rediscovered, because the source files remain in place. They should be a complementary durability feature, with a documented field list and loss model, rather than a promise that every Gallery feature survives a delete/re-add cycle.

## Proposed direction

Treat the two options as complementary, with **Option A as the primary feature**:

1. Add an explicit, admin-only “Transfer owner” operation that preserves the library and asset IDs. Do not model transfer as deleting and recreating the library or as editing `ownerId` through the general update endpoint.
2. Before implementation, settle the transfer policy for user-scoped metadata—especially people shared across assets outside the library, tags, favorites, mixed albums/stacks, and linked shared spaces—and the client-sync removal/backfill contract.
3. In a separate or follow-up slice, extend file-sidecar persistence for selected portable fields, starting with named face regions using the already-supported `RegionInfo` import format. Keep the supported field list and read-only limitations explicit.
4. Do not promise that “all metadata” is preserved by XMP alone. If full database teardown/re-import recovery is a requirement, design a Gallery-specific versioned catalog/manifest with stable asset and person identities and documented ownership, privacy, and conflict semantics.

This ordering addresses the account-switch use case without first requiring a new serialization format for every Gallery relationship. Sidecars then improve portability and resilience without being coupled to one user's database identity.

## Validation and test requirements

The transfer must be covered by tests that verify both retained metadata and access boundaries:

- A transfer keeps the same library and asset IDs, EXIF rows, supported edits, trims, face detections, and valid relationships.
- The target owner can read and manage the transferred assets; the former owner loses direct access unless a separate share still grants it.
- Tag, favorite, album, stack, person, and linked-space behavior matches the chosen policy, including assets/people that also exist outside the transferred library.
- The old and new users receive correct library/asset sync state after reconnecting from checkpoints created before transfer. Test the case where neither client is online during transfer.
- A failed or retried transfer cannot leave assets owned by a different account than their library or leave stale access grants.
- Sidecar round-trip tests cover supported metadata, named face regions, orientation-adjusted face geometry, external edits, read-only mounts, and repeated extraction without duplicate/stale faces.
- Deleting and re-adding a library demonstrates exactly which sidecar-backed fields are recovered and which Gallery-only relationships are not.

Any authorization gate added for transfer must follow the repository's enumerating authorization-test requirements. Any new sidecar schema or migration must preserve Gallery's revert-to-Immich and migration downgrade guarantees.

## Open decisions

- Should the previous owner retain any access to a transferred library, or is transfer always a full ownership handoff?
- Should the new owner inherit the former owner's favorites, tags, named people, person birthdays, and hidden-person settings?
- What should happen to albums/stacks and person groups that mix library assets with assets owned outside the library?
- Should shared-space links stay in place automatically, require confirmation, or be removed during transfer?
- Is the first release required to handle arbitrarily large libraries without a long transaction, and what is the acceptable period of transfer unavailability?
- Which face/person fields are in scope for portable sidecar write-back, and is a Gallery-specific stable person identity acceptable alongside standard XMP regions?
- Is delete/re-add recovery a supported requirement, or is the required workflow specifically an in-place owner transfer?