# RAW+Image Pairing for External Libraries

**Status:** Discussion draft — preferred direction captured; product policies remain open.

## Problem and scope

Photographers often keep a camera RAW file and its developed JPEG together in an external library, for example `photo.CR3` and `photo.jpg`, sometimes with `photo.xmp`. Gallery currently imports them as independent assets. That creates duplicate catalog entries and repeats face/person and other classification work.

The goal is for Gallery to treat a matched pair as **one logical image**, with one coherent metadata/classification identity and the JPEG as its normal display rendition, while keeping the RAW available as an alternate original. This is not a request for auto-stacking: a stack only groups assets visually and leaves their metadata, processing, and identity separate.

Initial scope is deliberately limited to an **opt-in setting on each external library**. Camera uploads and other upload paths are unchanged; users are not expected to upload RAW/JPEG pairs as part of this feature.

## Existing behavior and constraints

- The external-library scanner currently discovers and creates an asset per supported file. It retains each source path and filename, so files in the same directory with the same stem can be matched much more safely than uploads whose original directory may be lost.
- The server's authoritative image extension/MIME map and RAW category are in `server/src/utils/mime-types.ts`; `mimeTypes.getSupportedFileExtensions()` is based on all server-supported image and video extensions. The image map includes browser-supported formats and formats rendered by the server because browsers cannot display them directly. The current RAW-category extensions are `.3fr`, `.ari`, `.arw`, `.cap`, `.cin`, `.cr2`, `.cr3`, `.crw`, `.dcr`, `.dng`, `.erf`, `.fff`, `.iiq`, `.k25`, `.kdc`, `.mrw`, `.nef`, `.nrw`, `.orf`, `.ori`, `.pef`, `.psd`, `.raf`, `.raw`, `.rw2`, `.rwl`, `.sr2`, `.srf`, `.srw`, and `.x3f`. Treat this existing category as the pairing side: formats such as PSD may pair with JPEG when they represent alternate files for the same image.
- Browser-native image support is a smaller set in `web/src/lib/utils/asset-utils.ts`; it is not the list to use for pairing. The UI can display server-generated renditions of RAW, TIFF, HEIC, and other non-browser-native image formats.
- Sidecar discovery is per asset and checks both `photo.CR3.xmp` and `photo.xmp`-style names. With a pair, one same-stem XMP can otherwise be discovered by both assets. Pair-level sidecar ownership and write behavior must be explicit.
- Manual stacks do not solve the identity/classification problem. Existing research on auto-stacking is at [automatic-stacking research](research/03-automatic-stacking.md); burst stacking is separate and out of scope.
- Gallery is layered on Immich. Pair state belongs in fork-owned data/code; upstream asset rows and their meaning must remain intact. Stock Immich clients must continue to see the ordinary RAW and JPEG assets. Gallery client/server version mixing must also remain safe.
- A setting belongs to a specific external library, not a global preference: the same files can be paired in one library and remain independent in another.

## Approaches considered

| Approach | What it means | Decision |
| --- | --- | --- |
| **Auto-stack the pair** | Keep two independent assets and put them in one stack. | Rejected: the user explicitly wants metadata and classification collapsed too, not just the grid presentation. |
| **Suppress the RAW from Gallery views** | Hide the RAW in ordinary Gallery surfaces but otherwise leave it as an independent asset. | Insufficient alone: metadata and ML work remain split, and users need a clear route to the RAW. |
| **Skip importing the RAW** | Create only the JPEG asset. | Rejected: Gallery should retain and serve the RAW original. |
| **One Gallery logical image backed by both source assets** *(preferred)* | Retain the two upstream asset records for compatibility, but relate them in fork-owned state and expose one logical photo to Gallery-aware behavior. | Best fit: one Gallery identity with both originals retained; requires consistent projection and processing rules across Gallery surfaces. |

## Preferred direction

Add an opt-in **Pair RAW+Image** option to each external library, disabled by default. Keep the existing RAW and non-RAW asset rows so stock Immich continues to see both. Add a fork-owned relation that identifies the canonical non-RAW image and its RAW variant; Gallery-aware reads and clients treat the pair as one logical image. The RAW remains downloadable/viewable through an explicit variant action, not as a second timeline item.

Only pair with strong, deterministic evidence: same library and owner, same source directory and normalized stem, and exactly one supported photographic RAW plus one supported non-RAW image candidate. Any supported RAW extension may pair with any supported non-RAW image extension (for example CR3+JPEG or NEF+TIFF); pairing is not JPEG-specific. Match extensions case-insensitively. If there are multiple RAW or non-RAW candidates for one stem, leave those assets independent rather than guessing. The first implementation targets external-library scans and rescans, not upload-time pairing.

The pair should have one Gallery metadata/classification identity. Face/person recognition should run once against the canonical non-RAW image. Gallery should present and edit one set of tags, people, description, rating, and other metadata. Source EXIF may contribute fields to that logical metadata, but conflicts need a documented precedence rule. A same-stem XMP should be discovered and applied once to the pair, with a single defined write target.

Gallery's current image edits do **not** modify either original source file. Crop/rotate/mirror operations are stored as `asset_edit` actions; exposure/contrast/saturation/invert are stored in the fork-owned `asset_image_adjust` table. The server renders separate edited previews/fullsize outputs from the original asset. RAW assets already have server thumbnail generation, including embedded-preview extraction and decoding; the editor's transform/adjust operations can be applied to that rendered RAW-derived image. This is rendition editing, not RAW development (white balance, demosaicing controls, etc.) and does not rewrite a RAW file. For a pair, edits belong to the canonical non-RAW asset and affect Gallery's displayed rendition; the RAW download remains the untouched source.

For a paired item, normal download selects the non-RAW original. A separate **Download RAW** action is available when a RAW member exists, including from the asset menu. Deletion is explicit: ask whether to delete only the non-RAW rendition or both source assets. Deleting one member dissolves the logical pair and leaves the other asset in the database; it must not cascade-delete the other file.

Archive, favorite, album membership, and sharing apply to the logical image in Gallery and are mirrored to both underlying asset records. This keeps state consistent if the pair is later dissolved or an Immich client accesses either asset directly. These paired writes must be atomic or repaired safely on failure, and operations initiated through an Immich client against one asset need a defined synchronization path while the relation exists.

This must be a cross-surface behavior, not a timeline-only filter. Define how pairing is projected consistently in search, albums, people, map, memories, duplicates, statistics, shared spaces, asset detail, permissions, deletion, and sync. If a viewer is not allowed to see the RAW member, the logical image may still appear through its visible JPEG, but the RAW variant must not be disclosed or downloadable.

## Per-library setting and lifecycle

- Add an optional fork-owned per-library setting, default `false`, with a clear “Pair RAW+Image” label/help text in that external library's settings. Keep its API field additive and available only to Gallery-aware clients as required by compatibility policy.
- When enabled, a scan/rescan detects candidates, creates or refreshes pair relations, and backfills assets already in that library. It must not move, rename, rewrite, or delete source files.
- When disabled, stop applying logical-pair projection for that library and return to the two ordinary assets. Preserve user data and source files; define whether pair relations are retained for re-enabling or removed safely.
- Rescans must handle either file being added, removed, renamed, or going offline. A missing RAW should not hide the JPEG; a missing JPEG needs a decided fallback (show the RAW on its own or mark the logical image unavailable).
- An explicit unlink/“keep separate” override is needed if users can reject a detected match. The override must survive rescans while the files remain unchanged.

## Compatibility and layering

- Stock Immich clients continue to receive the two standard asset records and no unknown pair fields. A stock Immich server booting on the Gallery database must still find and serve both source assets.
- New Gallery clients treat the pair as one image only when the server advertises the capability/data. An older Gallery client against a newer server must remain safe (it may show two assets); a new client against an older server must behave normally because pair support is absent.
- Store the pair relation and per-library policy in fork-owned schema, keyed to upstream IDs. Register any new foreign keys to upstream keys as required by the repository's FK registry. Put implementation logic under the Gallery namespaces and keep upstream edits to narrow registration/hooks.
- A migration/backfill must be non-destructive and reversible. Reverting Gallery must leave the ordinary RAW/JPEG asset records usable by stock Immich.

## Decisions still needed

- **Resolved — formats:** Use Gallery's server-supported image formats, partitioned according to Gallery's existing RAW and non-RAW categories. Any supported non-RAW image format may pair with any supported RAW-category format, including PSD; extension comparison is case-insensitive.
- **Resolved — matching:** Exact same directory plus normalized same stem, within the same enabled library, is the intended match rule. Pair only when exactly one eligible RAW and one eligible non-RAW image share that stem; do not guess when there are multiples.
- What is the field-by-field precedence between XMP, JPEG EXIF, and RAW EXIF? Which source is updated when a user edits metadata in Gallery?
- **Resolved — display/download:** The non-RAW member is the display source and default download; provide a separate Download RAW action in the menu.
- **Resolved — delete:** Ask whether to delete the non-RAW member only or both. If a member disappears or is removed, retain the other asset row and dissolve the pair relation.
- **Resolved — logical operations:** Archive, favorite, album membership, and sharing apply to the logical image and are mirrored to both underlying assets. All Gallery and Immich write paths that can mutate either member must preserve or reconcile this invariant; partial mirror writes must be atomic or repaired safely.
- **Resolved — edits:** Existing image edits are non-destructive instructions plus generated derivatives, not source-file modifications. Store/apply pair edits to the canonical non-RAW member; RAW remains unchanged and downloadable. Gallery supports rendition transforms/adjustments on RAW-derived previews but not RAW development controls.
- When the setting is disabled, are pair links retained but ignored, or removed? What happens to user metadata/classification attached to both existing asset IDs during backfill and unlink?
- **Resolved — missing partner:** If one source file is removed, retain the other asset and remove the pair relation. Specify in implementation whether the one-file delete choice can be used in the UI for the same outcome.

## Out of scope

- Upload-time pairing, since this feature targets existing external libraries.
- Auto-stacking bursts, HDR brackets, or other related-photo groups.
- Deleting either source, guessing among ambiguous candidates, or treating unrelated same-name files in different directories/libraries as a pair.
