# External Library Owner Transfer and Metadata Portability

Date: 2026-11-05

## Status

Decisions recorded on 2026-11-07 from owner review (see [Decisions](#decisions)). Implementation has not started. The verification items in Decisions must pass first.

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

User-scoped associations follow the rules recorded in [Decisions](#decisions):

- Tags: map each transferred asset's tag links by name to the new owner's tags, creating missing tags and keeping hierarchy. Remove links from transferred assets to the former owner's tags. The former owner's tag rows stay unchanged for their other assets.
- Favorites: the former owner's `asset_favorite` rows are not touched, and the former owner keeps them visible through the retained shared-space access. The new owner's `asset.isFavorite` projection is recomputed from the new owner's overlay, so the new owner starts with no favorites.
- Albums, stacks, and people that mix library and non-library assets are split at the transferred assets, not moved whole.
- Faces: keep each `asset_face` row and its `shared_space_person_face` and `shared_space_person` links. Rebind only the owner-scoped person group.
- Sync grants: the owner grant is insert-only (`library_after_insert`), so finalization inserts the new owner's `library_user` row with the default `createId`. A `library_audit` row for the former owner is written only when no other access path remains, because `library_user_delete_after_audit` deletes the (userId, libraryId) row unconditionally and that row is shared by the owner and space paths. Finalization also bumps `library.updateId` and each transferred asset's `updateId`.

Large libraries use the resumable job in [Decisions](#decisions). A single transaction over every asset and face row is rejected: it holds locks and WAL for the whole transfer, and clients could observe a partial state.

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

1. Add an explicit, admin-only “Transfer owner” operation that preserves the library and asset IDs and runs as the job in [Decisions](#decisions). Do not model transfer as deleting and recreating the library or as editing `ownerId` through the general update endpoint.
2. Apply the policy in [Decisions](#decisions). Confirm the mixed album, stack, and person rules and pass the verification items before implementation.
3. In a separate or follow-up slice, extend file-sidecar persistence for selected portable fields, starting with named face regions using the already-supported `RegionInfo` import format. Keep the supported field list and read-only limitations explicit.
4. Do not promise that “all metadata” is preserved by XMP alone. If full database teardown/re-import recovery is a requirement, design a Gallery-specific versioned catalog/manifest with stable asset and person identities and documented ownership, privacy, and conflict semantics.

This ordering addresses the account-switch use case without first requiring a new serialization format for every Gallery relationship. Sidecars then improve portability and resilience without being coupled to one user's database identity.

## Validation and test requirements

The transfer must be covered by tests that verify both retained metadata and access boundaries:

- A transfer keeps the same library and asset IDs, EXIF rows, supported edits, trims, face detections, and valid relationships.
- The target owner can read and manage the transferred assets. The former owner has no ownership rights and reads them only through the retained shared space; removing that membership removes their access.
- Tag, favorite, album, stack, person, and linked-space behavior matches [Decisions](#decisions), including assets and people that also exist outside the transferred library. Mixed albums, stacks, and person groups are split, not moved whole.
- The old and new users receive correct library/asset sync state after reconnecting from checkpoints created before transfer. Test the case where neither client is online during transfer.
- A failed, interrupted, or retried job cannot leave assets owned by a different account than their library, leave stale access grants, or make the library visible to the new owner before finalization. A retry resumes from the last committed batch.
- Sidecar round-trip tests cover supported metadata, named face regions, orientation-adjusted face geometry, external edits, read-only mounts, and repeated extraction without duplicate/stale faces.
- Deleting and re-adding a library demonstrates exactly which sidecar-backed fields are recovered and which Gallery-only relationships are not.

Any authorization gate added for transfer must follow the repository's enumerating authorization-test requirements. Any new sidecar schema or migration must preserve Gallery's revert-to-Immich and migration downgrade guarantees.

## Decisions

Recorded on 2026-11-07 from owner review.

1. **Full ownership handoff.** The new owner owns the library, its assets, and the transferred metadata. The former owner's ownership ends at finalization.
2. **The former owner keeps read access through a shared space.** The former owner keeps their favorites by remaining a viewer of the library, not by copying favorites. The transfer creates a shared space owned by the new owner, links the library to it, and adds the former owner as a viewer. Consequence: the former owner can see every transferred asset, read-only, for as long as that space and membership exist. This is a visibility decision, not an ownership one, and the new owner must be told before the transfer.
3. **Tags.** Name-based mapping to the new owner's tags, as in Required behavior.
4. **Yes, these problems would follow unless the rules below apply.**
   - **Albums (privacy).** Album ownership lives in `album_user`, not on `album`. If an album the former owner owns keeps transferred assets, the former owner can share them with people the new owner never approved, and the new owner cannot remove them. Recommended rule: albums whose members are all library assets move to the new owner. Mixed albums stay with the former owner, transferred members are removed from them, and the removal is listed in the transfer report.
   - **Stacks (invariant).** `stack.ownerId` is separate from the owner of its assets. A mixed stack would have a primary asset owned by one account and the stack owned by another. Recommended rule: stacks whose members are all library assets move with their owner. Mixed stacks are split so the transferred members leave the stack.
   - **People (silent loss).** `asset_face.ownerId` is copied from the asset, while `person` is keyed by (ownerId, personGroupId). Any query that joins faces to people by owner drops transferred faces once their asset moves. Recommended rule: transferred faces move to a person group owned by the new owner. Faces on non-library assets stay with the former owner's person group.
   - **Later merges.** A former owner's person cannot merge with a transferred person across owners. The existing guard rejects this at write time, so an admin sees an error rather than a silent wrong merge.
   - **Shared-space dependency.** The former owner's view of transferred favorites depends on the space. Removing their membership or deleting the space hides those assets from them, but their `asset_favorite` rows survive. Unlinking the library from the space revokes their access through the existing shared-space library audit path.
   - **Repeat transfers.** `asset.isFavorite` is recomputed from the current owner's overlay at every transfer, so a second transfer does not carry a stale projection.
   The album, stack, and people rules are recommended defaults. They need confirmation before implementation.
5. **People links.** Each transferred face keeps its `asset_face` row ID. `shared_space_person_face` and `shared_space_person.faceAssetId` reference face IDs, so those links survive. The job rebinds each face's person group to the new owner's person and keeps any space link the face already has. The existing space/library face reconciliation is queued for linked spaces after finalization.
6. **Transfer runs as a job.** States: queued, running, finalizing, done, failed. While the job runs, the library is marked transferring, and scans, sidecar writes, and owner-scoped writes to its assets are blocked. Batches are keyed by asset ID, committed independently, and idempotent, so a retry resumes from the last batch. Finalization is the only step that changes visibility. It sets `library.ownerId`, inserts the new owner's `library_user` row, writes the former owner's `library_audit` row only when no other path remains, and bumps `library.updateId`. A failure before finalization leaves the library with its former owner. A reverse run restores that owner, since asset IDs are unchanged.

Verify before implementation. Decisions 2 and 5 depend on these:

- Whether a shared-space viewer can write their own `asset_favorite` overlay row. If not, decision 2 cannot keep favorites live and needs a permission change.
- Whether the `asset_favorite` triggers fire on the bulk `asset.ownerId` update, and whether they recompute the projection in the right order.
- Whether the shared-space `library_user` insert triggers use `ON CONFLICT`, so an insert over an existing row is safe.
- Which sync path tells the former owner's clients that owned assets are gone when their access continues through the space.

## Open items

- Acceptable unavailability window and batch size for the job.
- Whether birthdays and hidden-person settings move with transferred person groups. Decision 2 settled favorites only.
- Which face/person fields are in scope for portable sidecar write-back, and whether a Gallery-specific stable person identity is acceptable alongside standard XMP regions.
- Whether delete/re-add recovery is a supported requirement, or the required workflow is specifically an in-place owner transfer.