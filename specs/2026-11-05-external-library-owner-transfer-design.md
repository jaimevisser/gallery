# External Library Owner Transfer and Metadata Portability

Date: 2026-11-05

## Status

Decisions recorded on 2026-11-07 from owner review (see [Decisions](#decisions)), including the admin-facing wording in Decision 7. Decision 5 (people) and Decision 7 were amended on 2026-11-08: transfers copy people rather than moving person groups, and the admin gets an optional people-merging step before confirming. Implementation has not started. Admin flow mockups: [library-owner-transfer.html](mockups/2026-11-05-external-library-owner-transfer/library-owner-transfer.html). The former owner's fork-client sync path is read on code and converges through a bumped `asset.updateId`; stock-client staleness is accepted (see the verify list), and a pre-transfer checkpoint test remains open.

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
- Albums and stacks that mix library and non-library assets are split at the transferred assets, not moved whole. People are copied or merged, not moved (Decision 5).
- Faces: keep each `asset_face` row and its `shared_space_person_face` and `shared_space_person` links. Point each transferred face at a person owned by the new owner: a copy of its former person (name and birthday), or an existing new-owner person the admin merged into before confirming (see Decision 5).
- Sync grants: the owner grant is insert-only (`library_after_insert`), so finalization inserts the new owner's `library_user` row with the default `createId`. A `library_audit` row for the former owner is written only when no other access path remains, because `library_user_delete_after_audit` deletes the (userId, libraryId) row unconditionally and that row is shared by the owner and space paths. Finalization also bumps `library.updateId`, each transferred asset's `updateId`, and each transferred face's `updateId`. The face stream is cursored by `asset_face.updateId` and has no backfill, so without the face bump the new owner's clients never receive those faces.

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
2. Apply the policy in [Decisions](#decisions) and pass the open verification item before implementation.
3. In a separate or follow-up slice, extend file-sidecar persistence for selected portable fields, starting with named face regions using the already-supported `RegionInfo` import format. Keep the supported field list and read-only limitations explicit.
4. Do not promise that “all metadata” is preserved by XMP alone. If full database teardown/re-import recovery is a requirement, design a Gallery-specific versioned catalog/manifest with stable asset and person identities and documented ownership, privacy, and conflict semantics.

This ordering addresses the account-switch use case without first requiring a new serialization format for every Gallery relationship. Sidecars then improve portability and resilience without being coupled to one user's database identity.

## Validation and test requirements

The transfer must be covered by tests that verify both retained metadata and access boundaries:

- A transfer keeps the same library and asset IDs, EXIF rows, supported edits, trims, face detections, and valid relationships.
- The target owner can read and manage the transferred assets. The former owner has no ownership rights and reads them only through the retained shared space; removing that membership removes their access.
- Tag, favorite, album, stack, person, and linked-space behavior matches [Decisions](#decisions), including assets and people that also exist outside the transferred library. Mixed albums and stacks are split, not moved whole, and people are copied or merged rather than moved.
- Transferred faces are copied to new-owner persons with the source name and birthday (and pet type and species). Hidden, favorite, and identity links are not copied. A confirmed merge reuses the chosen new-owner person. The former owner's person groups and their remaining faces are unchanged, and shared-space person rows are unchanged.
- Shares on the former owner's person groups are unchanged, including a share to the new owner. Copied persons have no shares, so the self-share check is never reached.
- The old and new users receive correct library/asset sync state after reconnecting from checkpoints created before transfer. Test the case where neither client is online during transfer. The new owner's clients must also receive the transferred faces, which requires the face `updateId` bump.
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
   - **People (silent loss).** `asset_face.ownerId` is copied from the asset, while `person` is keyed by (ownerId, personGroupId). Any query that joins faces to people by owner drops transferred faces once their asset moves. Rule (amended 2026-11-08, Decision 5): transferred faces are re-pointed to a person owned by the new owner. Faces on non-library assets stay with the former owner's person group.
   - **Later merges.** A former owner's person cannot merge with a transferred person across owners. The existing guard rejects this at write time, so an admin sees an error rather than a silent wrong merge.
   - **Shared-space dependency.** The former owner's view of transferred favorites depends on the space. Removing their membership or deleting the space hides those assets from them, but their `asset_favorite` rows survive. Unlinking the library from the space revokes their access through the existing shared-space library audit path.
   - **Repeat transfers.** `asset.isFavorite` is recomputed from the current owner's overlay at every transfer, so a second transfer does not carry a stale projection.
   The owner confirmed the album, stack, and people rules on 2026-11-07. The people rule was amended on 2026-11-08 (Decision 5).
5. **People.** Each transferred face keeps its `asset_face` row ID, so `shared_space_person_face` and `shared_space_person.faceAssetId` links survive. Each face is re-pointed to a person owned by the new owner. The former owner's person groups are not moved: they keep their faces on assets outside the library, and the former owner keeps seeing those people there. The existing space/library face reconciliation is queued for linked spaces after finalization. (Amended 2026-11-08: this replaces the earlier rule that moved whole person groups.)

   Before confirming, the admin sees a people-merging step (Decision 7). For each person with faces in the library, it proposes an existing new-owner person as a merge target when the names are identical, or when both users are in a shared space that links the library and that space shows the same person. The admin confirms or changes each merge; this is optional. A confirmed merge re-points the transferred faces to that person, and its own name, birthday, and settings are left as they are. Any other person gets a copy with the former person's name and birthday, and pet type and species for pets. Hidden, favorite, and identity-link settings are not copied, so existing identities of the new owner are not affected. Shares on the former owner's person groups are unchanged. The transfer creates no shares. The space-suggestion rule is accepted as written (2026-11-08). Later merges use the existing same-owner merge (2026-11-08). The transfer-time merge is a one-off repoint within this transfer, not that later merge.
6. **Transfer runs as a job.** States: queued, running, finalizing, done, failed. While the job runs, the library is marked transferring, and scans, sidecar writes, and owner-scoped writes to its assets are blocked. Batches of 1,000 assets are keyed by asset ID, committed independently, and idempotent, so a retry resumes from the last batch. Reads continue during the run, and visibility changes only at finalization. The owner accepted this window on 2026-11-07. Finalization is the only step that changes visibility. It sets `library.ownerId`, inserts the new owner's `library_user` row, writes the former owner's `library_audit` row only when no other path remains, and bumps `library.updateId`. A failure before finalization leaves the library with its former owner. A reverse run restores that owner, since asset IDs are unchanged.
7. **What the admin is told.** The confirmation dialog states the following before the admin confirms. Plain language; no storage terms. The English text is the source for the overlay strings in the nine maintained locales.

   > **Transfer "{library}" to {new owner}?**
   >
   > - {new owner} becomes the owner of every photo and video in this library, with their tags, and any albums or stacks that contain only these photos.
   > - {new owner} gets copies of the people and pets in these photos, with their names and birthdays. Hidden and favorite settings are not copied. Before you confirm, you can merge each one into a person {new owner} already has. {former owner} keeps their own people and the photos of those people in their other libraries.
   > - A shared space owned by {new owner} is created so {former owner} can keep viewing these photos, read-only. {former owner}'s favorites stay with them. They lose access when they leave that space or the space is deleted.
   > - {former owner} may keep seeing these photos as their own in the official Immich app until they reinstall it. Gallery apps update on their own.
   > - Albums and stacks that also contain photos outside this library stay with {former owner}. The transferred photos are removed from them.
   > - The library cannot be scanned, and its photos cannot be edited, until the transfer finishes. If the transfer is interrupted, it resumes. If it fails, the library stays with {former owner}.
   > - When it finishes, you get a report of what moved, what was removed from mixed albums and stacks, and which people were merged or copied.

Verify before implementation. Decisions 2 and 5 depend on these:

- Shared-space viewer favorite writes: passes on code. The favorite write requires only `AssetRead`, which a space viewer holds. The RBAC oracle should confirm the viewer role.
- `asset_favorite` triggers on the bulk `asset.ownerId` update: do not fire, because the forward trigger acts on overlay rows and the reverse trigger acts only on `isFavorite` writes. The job must recompute `asset.isFavorite` from the new owner's overlay explicitly (Decision 4). Open detail: the reverse trigger's event clause is not yet read.
- Shared-space `library_user` inserts: passes. All library-user inserts use `ON CONFLICT DO NOTHING`.
- Sync path for the former owner's clients when access continues through the space: read on code, a test from a pre-transfer checkpoint is still required. Findings:
  - `AssetSync.getUpserts` (`server/src/repositories/sync.repository.ts`, ~383) filters `ownerId = userId`, so the former owner's own-assets stream stops returning transferred rows.
  - `AssetSync.getDeletes` reads `asset_audit`, which only `asset_delete_audit` writes, on physical DELETE. An owner change emits no audit row, so no delete reaches the former owner through the own-assets stream.
  - `LibraryAssetSync.getUpserts` (`server/src/gallery/repositories/gallery-sync.repository.ts`, ~569) streams library assets for `accessibleLibraries` with `asset.updateId` as the cursor, and the `libraryId`/ownership predicate keeps the space-link path. A bumped `asset.updateId` therefore re-delivers each transferred asset with the new `ownerId`. Fork clients apply it through `updateAssetsV1`, which writes `ownerId` (`mobile/lib/infrastructure/repositories/sync_stream.repository.dart`, ~255/297). Convergence for fork clients depends on that bump.
  - Stock clients have no library stream. They keep the row as an owned asset with a stale `ownerId` until a full resync: no delete and no re-send. This is a gap to decide on, not a bug in the code path.
  - Do not fix the stock gap by writing `asset_audit` rows for the former owner. Fork clients would delete the row, `deleteAssetsV1` (`sync_stream.repository.dart`, ~227) would cascade face rows locally, and the library upsert would re-add the asset without them. Faces and album links would then need their own `updateId` bumps. Decision: accepted. Stock clients keep transferred assets as owned until a resync or reinstall. Transfers are rare, so no fix is planned. An optional tombstone that would close this gap is described in [Option C](#option-c-stock-client-tombstone-deferred); it is deferred.
  - Still to test: a fork client whose checkpoint predates the transfer, and whether the former owner's face stream drops transferred faces.

## Implementation spec

Status: proposed 2026-11-08 for review. No code exists. Code references were read on 2026-11-08 and must be re-checked when implementation starts. The items under [Decisions needed](#decisions-needed-before-coding) change Decisions 5 and 6 and need the owner's answer before coding.

### Scope

An admin-only transfer of one external library's owner, per Decisions 1 to 7. The change covers the server, one fork migration, and the web admin UI. Mobile is unchanged, because libraries are managed only in the web admin. The stock-client tombstone (Option C) is out of scope.

### Code facts this builds on

- `LibraryController` (`server/src/controllers/library.controller.ts`) serves `/libraries` behind `@Authenticated({ admin: true })`. `UpdateLibraryDto` has no owner field, so `PUT /libraries/:id` cannot change the owner.
- `LibraryService.delete` soft-deletes the library, then runs `JobName.LibraryDelete`, which pages library assets with `JOBS_LIBRARY_PAGINATION_SIZE`. This is the batching pattern to reuse for planning.
- `library.updateId` (`server/src/schema/tables/library.table.ts`) is an `UpdateIdColumn`, so bumping it re-delivers the library to clients.
- `library_user` is keyed `(userId, libraryId)` with a `createId`. Inserts use `ON CONFLICT DO NOTHING`.
- Shared-space entry points are `SharedSpaceService.create` (`server/src/services/shared-space.service.ts`, ~254), `addMember` (~579), and `linkLibrary` (~1027).
- Fork controllers, services, and repositories register in `galleryControllers`, `galleryServices`, and `galleryRepositories` in `server/src/gallery/index.ts`.

### Data model

All new objects are fork tables keyed to Immich ids (AGENTS rule 1). The `library` table is not changed beyond the `updateId` bump it already supports.

`library_owner_transfer` holds one row per transfer:
- `id` uuid primary key; `libraryId`, `fromOwnerId`, and `toOwnerId`, which are fork foreign keys onto `library.id` and `user.id`.
- `state` text: `queued`, `running`, `finalizing`, `done`, `failed`, or `cancelled`, enforced by a check constraint.
- `spaceId` uuid (null until cutover), `error` text, `report` jsonb (null until `done`), `createdAt`, and `updatedAt`.
- A partial unique index on `libraryId` where `state` is `queued`, `running`, or `finalizing`. One active transfer per library.

`library_owner_transfer_person` holds the planned person mapping, with primary key `(transferId, sourcePersonId)`:
- `action` (`copy` or `merge`) and `targetPersonId`. Copies are created during planning. A merge targets an existing new-owner person.
- The primary key lets a retried planning step find its rows instead of creating duplicates.

`library_owner_transfer_tag` holds the planned tag mapping, with primary key `(transferId, sourceTagId)` and `targetTagId`. A target tag is the new owner's tag with the same name, or a new one.

The fork foreign keys (`libraryId`, `fromOwnerId`, `toOwnerId`, `sourcePersonId`, `targetPersonId`, `sourceTagId`, `targetTagId`) are listed in `server/src/schema/gallery-upstream-fks.json`.

### Preview and planning

These steps make no visible change.

`GET /libraries/:id/owner-transfer/preview?toOwnerId=` is read-only and returns:
- Counts of assets, faces, and tag links on library assets.
- One people entry per source person with faces in the library: `sourcePersonId`, name, birthday, pet type and species for pets, face count, and merge candidates.
- Merge candidates (Decision 5): a new-owner person with the same name, or a person shown in a shared space that links the library and has both users as members.
- Mixed albums and stacks that will be split, listed by name.
- The tag mapping by name.

`POST /libraries/:id/owner-transfer` takes `{ toOwnerId, merges: [{ sourcePersonId, targetPersonId }] }`. It creates the row in `queued` and returns `202` with the transfer. The server recomputes the plan on start. It rejects a merge whose target is not a person of `toOwnerId` or is hidden. A second active transfer for the library returns `409`.

The planning step creates the copied persons and the target tags, writes the mapping tables, and sets the state to `running`. A crash during planning leaves mapping rows and unused copies. A retry reuses them through the primary keys. On `failed`, the job deletes the copies the plan created.

`GET /libraries/:id/owner-transfer` returns the latest transfer: `state`, `error`, `report`, and `spaceId` once set. `DELETE /libraries/:id/owner-transfer` cancels a transfer in `queued`. It returns `409` for `running` and `finalizing`.

### Cutover

Cutover is one transaction. It does these steps in order:
1. Sets `library.ownerId = toOwnerId` and bumps `library.updateId`.
2. Sets `asset.ownerId` on library assets and bumps their `asset.updateId`.
3. Re-points each face on a library asset whose person is a mapped source to the target person, using the person column on `asset_face`. Bumps `asset_face.updateId` for every face on a library asset. The `ownerId` trigger copies the owner onto faces, and the face stream has no backfill, so the bump is required.
4. Replaces tag links on library assets: inserts the target tag link and deletes the source tag link. Bumps `asset.updateId`.
5. Recomputes `asset.isFavorite` on library assets from the new owner's `asset_favorite` overlay rows.
6. Splits mixed albums and stacks by removing the transferred members. Albums and stacks made only of library assets move to `toOwnerId`.
7. Inserts the new owner's `library_user` row with the default `createId`. The former owner's `library_user` row stays, because the space in step 8 grants them access through it. No `library_audit` row is written at cutover.
8. Creates the shared space owned by `toOwnerId`, links the library, and adds the former owner as a viewer.
9. Writes `report` and sets `state` to `done`.

If any step fails, the transaction rolls back. The library stays with the former owner, and the state becomes `failed` with `error` set. Nothing is visible in between.

After cutover, the former owner's own-assets stream stops returning transferred assets, because `AssetSync.getUpserts` filters on `ownerId`. The library stream delivers them through the space instead.

### Write blocking

While a transfer is `queued`, `running`, or `finalizing`, these requests return `409` for the library and its assets:
- Scan (`POST /libraries/:id/scan`), library update, and library delete.
- Asset update and delete for library assets, including the bulk endpoints.
- Tag, album, and stack membership edits that include library assets.
- Sidecar writes for library assets.

Jobs queued before the transfer can still run during it. The scan and sidecar job handlers check the same state when they run. One helper, `assertLibraryNotTransferring(libraryId)`, in the gallery module serves both the routes and the jobs. The final route list comes from a grep of the write paths during implementation and is reviewed in the PR.

### Authorization

Every endpoint is admin-only. The matrix test follows AGENTS rule 7. Actors are admin, a non-admin owner of the library, a non-admin space member, and an unauthenticated caller. Operations are preview, start, status, and cancel. Library states are none, queued, running, done, and failed. Expected outcomes come from a truth table in this spec, not from the production helper. Each denied request checks that no row was written. The admin check is mutation-checked once by removing it.

### Report

Written at cutover:

`{ fromOwnerId, toOwnerId, spaceId, assetsMoved, facesRepointed, personsCopied, personsMerged, tagLinksReplaced, favoritesRecomputed, albumsSplit: [{ name, removedCount }], stacksSplit: [{ name, removedCount }], completedAt }`

### Sync

No new sync stream. Expected results for fork clients follow the verify list above. The former owner's own-assets stream stops returning transferred assets. The library stream delivers them through the space with the new `ownerId`. The new owner receives the library and its assets through `library_user`. Stock clients see the accepted gap.

### Migration and revert

- One fork migration in `server/src/schema/migrations-gallery/`, with a round timestamp above the newest upstream and fork migrations. Check the current newest at implementation time; the newest fork file seen on 2026-11-08 is `1799700000000`. The proposed name is `1800000000000-AddLibraryOwnerTransfer`.
- `down()` refuses while any transfer is active, then drops the tables. The ownership changes stay in upstream tables that an older Gallery already reads, so there is no data to move back. This departs from the AGENTS rule that `down()` moves data back, so it needs the owner's approval (see Decisions needed, item 4).
- The new tables are added to `scripts/revert-to-immich.sql`. `server/src/schema/revert-to-immich.spec.ts` must pass.
- Upstream files touched: `JobName` in `server/src/enum.ts` (one registration line), `scripts/revert-to-immich.sql`, and `server/src/schema/gallery-upstream-fks.json`. The preflight diff against the upstream tag reports each one.

### Web

- The entry point is one line in the upstream library admin page that renders a fork component from `web/src/lib/gallery/`.
- The component follows the five steps in the mockup. Step 3's merge choices come from the preview response. Strings come from the nine-locale overlay, using the Decision 7 text.
- Regenerate the SDK after the server build (`mise run open-api`).
- Component tests cover step gating, merge toggles, disabled Merge rows with no candidate, and Cancel versus Back.

### Tests

Unit tests: the merge-candidate rule, state transitions, the guard helper, and mapping idempotence.

Medium tests, against a real database:
- Happy path: library and asset IDs are unchanged. EXIF, edits, trims, and asset-to-face links are intact. Faces point to copies with the source name and birthday. Pet type and species are copied. Hidden, favorite, and identity settings are not copied. A merged person keeps its own settings.
- The former owner's faces on non-library assets and their persons are unchanged. Shares on the former owner's person groups are unchanged. Copies have no shares.
- Tags are remapped by name. Tags on non-library assets are untouched.
- `asset.isFavorite` is recomputed from the new owner's overlay. The former owner's `asset_favorite` rows are untouched.
- Mixed albums and stacks are split. Albums and stacks made only of library assets move.
- The shared space is created. The former owner reads through it. Removing their membership removes access.
- A failure at each cutover step leaves no visible change, sets `failed`, and removes the copies.
- A crash during planning, then a retry, creates no duplicate copies.
- A reverse transfer restores ownership. A repeat transfer recomputes the favorite projection.
- A scan or asset write during `running` is rejected. A queued scan job checks the guard when it runs.
- Sync: pre-transfer checkpoints for both owners, from the open verification list. The new owner receives transferred faces through the face `updateId` bump.
- The authorization matrix described above.

Migration tests: up and down on a seeded database, and `down()` refuses during an active transfer.

Scale check: measure cutover on a seeded 100,000-asset library before the design is final (Decisions needed, item 1).

### Decisions needed before coding

1. **Cutover in one transaction, not batches.** Batched visible writes break the Decision 6 rule that visibility changes only at finalization. The former owner's own-assets stream filters on `ownerId`, so each batch would remove assets from it mid-run. Recommendation: one transaction, after the scale check. This amends the batch wording in Decision 6.
2. **Copied persons exist before cutover.** The new owner's people list can show copies with no visible faces during a run. Recommendation: accept this. The alternative is creating copies inside the cutover transaction, which moves planning into it.
3. **Cancel only in `queued`.** Recommendation: yes. A running cutover cannot be cancelled, only completed or failed.
4. **`down()` drops the tables instead of moving data back.** Recommendation: yes, with the refusal described above. This departs from the AGENTS rule.
5. **Permission.** Either reuse `LibraryUpdate` with `admin: true`, or add `LibraryOwnerTransfer` to `Permission`. A new permission adds an entry to the access-oracle snapshot. Recommendation: reuse `LibraryUpdate`.

### Verification before coding

- The pre-transfer checkpoint sync test for both owners, from the verify list.
- The reverse trigger's event clause, from the verify list.
- Whether `SharedSpaceService.create`, `linkLibrary`, and `addMember` can run inside the cutover transaction, and which `AuthDto` the job builds for `toOwnerId`.
- Whether the person create path creates a `person_group` row, which the copies need.

### Phases

1. Migration, tables, FK registry, revert script, and the state machine, each with tests.
2. Preview and planning, with the copy and merge rules.
3. Cutover and report, with sync and authorization tests.
4. Write guard across routes and jobs.
5. Web admin flow, component tests, SDK regeneration, and nine-locale strings.

Cutover is atomic, so no phase leaves a half-visible state.

## Option C: stock-client tombstone (deferred)

This option would stop stock Immich clients from keeping transferred assets as owned. It is not part of the accepted decision above. Deferred on 2026-11-08: the transfer ships with the documented stock gap. Revisit only if stock users report stale owned photos.

### Mechanism

- A fork table, `asset_owner_transfer_tombstone`, holds one row per transferred asset and former owner: `assetId`, `formerOwnerId`, and an `id` that is a UUIDv7 in the same watermark domain as `asset_audit.id`.
- Finalization writes the rows in the same transaction as the owner change. They are not `asset_audit` rows, so fork clients never read them.
- `syncAssetsV2` (`server/src/services/sync.service.ts`, ~318) already sends `AssetDeleteV1` before upserts. For clients where `isForkAwareClient` is false, the tombstone rows for that user are merged into the delete stream. The stock client removes the rows before any upsert arrives.
- A reverse transfer deletes the tombstones for the returning owner in the same finalization, so a stale delete cannot remove an asset that is owned again.
- Cleanup is added to `cleanupAuditTables` (`server/src/gallery/utils/gallery-sync.streams.ts`, ~1211), with the same retention as `asset_audit`.
- Schema: a fork migration with a `down()`. The `assetId` foreign key is a fork key onto an upstream key, so it is listed in `server/src/schema/gallery-upstream-fks.json`.

### Tests required

- A stock client whose checkpoint predates the transfer receives the delete and no longer holds the rows.
- A fork client receives no tombstone for the same transfer.
- A reverse transfer re-delivers the asset to the returning owner without a later delete.
- A client past retention gets a full reset and holds only its current assets.

### Unverified

- Whether the stock app's backup re-checks device photos against the server and re-uploads them as new assets for the former owner. A tombstone would not prevent this.
- Whether thumbnails cached on disk by the stock app are removed with the rows.
- Whether the tombstone stream can share one checkpoint with `asset_audit` ids without a separate watermark.

### Trade-off

The option adds a table, a migration, a branch in the stock sync path, cleanup, and tests. It removes the stale-owned rows only for transfers made after it ships, and it does not cover backup or cached files. Sidecar work (Option B) does not depend on it. Decision (2026-11-08): finish the transfer with the documented stock gap and defer this option.

## Open items

- Stock-client tombstone (Option C) is deferred (2026-11-08). Only a stock-user report would reopen the stock-client decision above.
- Sidecar write-back and delete/re-add recovery are out of scope for this transfer (answered 2026-11-08). They remain separate work under Option B.
- Pet type and species on a copied pet are copied (confirmed 2026-11-08).
- The space-suggestion rule for merge candidates is accepted as written (2026-11-08).