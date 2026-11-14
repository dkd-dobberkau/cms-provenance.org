# Media Provenance (Case 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** The signed Record of every page lists the images, videos and audio files used on it, with stable identifier, SHA-256 of the original, creator, copyright, licence, source and IPTC digital source type, kept current when files or their metadata change.

**Architecture:** The core extension stores identity data per file (`sys_file` columns plus a hash history table), filled by FAL index events and a backfill command. At render time `JsonLdHeaderData` reads the page's file references, adds `image`/`video`/`audio` to the Record and tags the page cache per file; metadata saves and file updates flush those tags, so the page is re-rendered and re-signed. The Pages submodule reports incomplete media. The specification gains the optional members.

**Tech Stack:** PHP 8.2+, TYPO3 13.4/14 (FAL, PSR-14 events, DataHandler hooks, `frontend.cache.collector`), EXT:filemetadata, Doctrine DBAL, Fluid, typo3/testing-framework ^9.7.

**Spec:** `docs/superpowers/specs/2026-11-14-media-provenance-design.md` (in `cms-provenance.org`).

## Global Constraints

- PHP `^8.2`; TYPO3 `^13.4 || ^14.0`. Repos: core `~/Versioncontrol/local/typo3-provenance`, spec `~/Versioncontrol/local/content-provenance-spec`, conformance `~/Versioncontrol/local/content-provenance-conformance`, site `~/Versioncontrol/local/cms-provenance.org`. Work on `main`, push after each task, CI must stay green (PHP 8.2/8.4 × TYPO3 13.4/14).
- `provenance_signing` is not changed: `SignedRecordRepository::save()` already upserts per page and language, so a re-rendered page replaces its Audit Endpoint row.
- Missing media data never changes `cpsConformance` and never appears in `ConformanceLevel::missing()`.
- No database writes while rendering a page; identifiers and hashes are written only by FAL events, DataHandler hooks and the backfill command.
- Unknown values are omitted, never emitted empty or guessed; no default digital source type.
- Only `image/*`, `video/*`, `audio/*` files; online media helpers (extensions from `OnlineMediaHelperRegistry::getSupportedFileExtensions()`) are skipped; only references from `tt_content`.
- `creator` is emitted as `{"name": …}` without `@type`.
- Digital source type IRI prefix: `https://cv.iptc.org/newscodes/digitalsourcetype/`.
- English labels in XLIFF; code and docs in English. Commit messages end with the session's attribution lines.

## Review Focus

1. The same file used twice on a page (two references, or two content elements) → listed once, at its first position. Test in Task 4.
2. A file re-indexed with unchanged bytes (`sha1` unchanged) → no new history row, no cache flush. Test in Task 2.
3. A file whose storage is offline or whose bytes cannot be read → indexing still succeeds, the error is logged, the file is listed without `sha256`. Test in Task 2.
4. A hidden content element or a hidden file reference → its file is not listed. Test in Task 4.
5. A page in language 1 with translated file metadata → translated `creator`, `license` and `digitalSourceType` from the default language. Test in Task 4.

---

### Task 1: Storage for media identity and metadata fields

Repository: core.

**Files:**
- Modify: `composer.json` (require `typo3/cms-filemetadata: ^13.4 || ^14.0`), `ext_emconf.php` (depends `filemetadata`), `ext_tables.sql`
- Create: `Configuration/TCA/Overrides/sys_file_metadata.php`
- Modify: `Resources/Private/Language/locallang_db.xlf`
- Test: `Tests/Functional/DataHandling/FileMetadataTest.php`, `Tests/Functional/DataHandling/Fixtures/FileMetadata.csv`

**Interfaces:**
- Produces: columns `sys_file.tx_provenance_media_uri varchar(255) DEFAULT '' NOT NULL`, `sys_file.tx_provenance_sha256 char(64) DEFAULT '' NOT NULL`, `sys_file.tx_provenance_sha256_of char(40) DEFAULT '' NOT NULL`; table `tx_provenance_media_hash` (`uid`, `media_uri varchar(255)`, `sha256 char(64)`, `valid_from int`, `valid_until int DEFAULT NULL`, keys on `media_uri` and `sha256`), defined only in `ext_tables.sql`, no TCA; TCA fields `sys_file_metadata.tx_provenance_license` (`type: link`, `allowedTypes: ['url']`, `l10n_mode: exclude`) and `sys_file_metadata.tx_provenance_digital_source_type` (`type: select`, `renderType: selectSingle`, first item empty, `l10n_mode: exclude`).
- Select items: the IPTC Digital Source Type terms from `https://cv.iptc.org/newscodes/digitalsourcetype/`, stored as the bare term (`digitalCapture`, …); at least `digitalCapture`, `compositeWithTrainedAlgorithmicMedia`, `trainedAlgorithmicMedia`; labels from IPTC's English names.
- Both fields shown in the metadata form of all file types, in a palette "Provenance" after `creator`/`copyright`.

- [ ] **Step 1: Failing test** `FileMetadataTest` (`$coreExtensionsToLoad = ['filemetadata']`): `licenseAndDigitalSourceTypeAreStored` saves metadata uid 1 via DataHandler with `tx_provenance_license = 'https://creativecommons.org/licenses/by/4.0/'` and `tx_provenance_digital_source_type = 'digitalCapture'`, asserts both in the DB row; `nonUrlLicenseIsRejected` with `'CC BY'` asserts the stored value is `''`.
- [ ] **Step 2:** `composer test:functional` → FAIL (unknown columns).
- [ ] **Step 3:** Add composer/emconf dependency, SQL, TCA override, labels.
- [ ] **Step 4:** `composer test` → PASS.
- [ ] **Step 5:** Commit `feat: add licence, digital source type and media identity storage`.

### Task 2: Media identity service

Repository: core.

**Files:**
- Create: `Classes/Media/MediaIdentity.php`
- Test: `Tests/Functional/Media/MediaIdentityTest.php`, `Tests/Functional/Media/Fixtures/Files.csv` (storage uid 1, local driver, `basePath: fileadmin/`; files created in `setUp()` under the instance's `fileadmin/`)

**Interfaces:**
- Consumes: `DocumentIdentifierGenerator::generate(): string` (existing), Task 1 columns.
- Produces: `MediaIdentity::update(int $fileUid): bool` — assigns `tx_provenance_media_uri` if empty; when `sys_file.sha1 !== tx_provenance_sha256_of`, computes SHA-256 over the file bytes (`File::getForLocalProcessing(false)` + `hash_file`), writes `tx_provenance_sha256` and `tx_provenance_sha256_of`, closes the open history row (`valid_until = now`) and inserts a new one. Returns true when the identifier or hash changed. Skips non-media and online-media files (returns false). Read errors are logged (`LoggerInterface`) and return false without changing the hash.
- Produces: `MediaIdentity::retire(int $fileUid): void` — closes the open history row of the file. `MediaIdentity::isMedia(array $fileRow): bool` — shared MIME/online-media check (`mime_type`, `extension`), used by Tasks 3 and 4.

- [ ] **Step 1: Failing tests:**
  - `assignsIdentifierAndHash`: after `update(1)`, `tx_provenance_media_uri` matches `/^urn:uuid:[0-9a-f-]{36}$/`, `tx_provenance_sha256 === hash('sha256', <file bytes>)`, one history row with `valid_until IS NULL`; returns true.
  - `keepsIdentifierWhenBytesChange`: change bytes and `sha1`, `update(1)` → same URI, new hash, two history rows, the old one closed.
  - `unchangedBytesWriteNothing` (Review Focus 2): second `update(1)` returns false, still one history row.
  - `unreadableFileIsLoggedAndKeepsOldHash` (Review Focus 3): delete the file on disk, change `sha1` → returns false, hash unchanged, no exception.
  - `nonMediaFilesAreSkipped`: a `application/pdf` file → returns false, no URI.
  - `retireClosesHistory`.
- [ ] **Step 2:** `composer test:functional` → FAIL.
- [ ] **Step 3:** Implement `MediaIdentity` (final readonly, autowired).
- [ ] **Step 4:** `composer test` → PASS.
- [ ] **Step 5:** Commit `feat: assign media identifiers and keep a hash history`.

### Task 3: Keep identities current (FAL events, backfill)

Repository: core.

**Files:**
- Create: `Classes/EventListener/UpdateMediaIdentity.php`, `Classes/Command/BackfillMediaCommand.php`
- Test: `Tests/Functional/EventListener/UpdateMediaIdentityTest.php`, `Tests/Functional/Command/BackfillMediaCommandTest.php`

**Interfaces:**
- Consumes: `MediaIdentity::update()`, `::retire()`; `MediaCacheTags` from Task 6 is **not** used here (Task 6 adds the flush to this listener).
- Produces: listener methods for `AfterFileAddedToIndexEvent` (`getFileUid()`), `AfterFileUpdatedInIndexEvent` (`getFile()->getUid()`) → `update()`; `AfterFileRemovedFromIndexEvent` (`getFileUid()`) → `retire()`. One `#[AsEventListener]` per method, identifiers `provenance/media-identity-added|updated|removed`.
- Produces: command `provenance:backfill-media` with `--dry-run`; processes `sys_file` rows with `missing = 0` and (`tx_provenance_media_uri = ''` or `tx_provenance_sha256_of <> sha1`), skipping non-media via `MediaIdentity::isMedia()`; prints the count updated.

- [ ] **Step 1: Failing tests:** adding a file through `ResourceStorage::addFile()` yields a URI and hash; `replaceFile()` with other bytes yields a new hash and a closed history row; `deleteFile()` closes the history row; backfill on a fixture with two unidentified files reports `2` and fills both; `--dry-run` writes nothing.
- [ ] **Step 2:** `composer test:functional` → FAIL.
- [ ] **Step 3:** Implement listener and command (pattern: `BackfillCommand`).
- [ ] **Step 4:** `composer test` → PASS.
- [ ] **Step 5:** Commit `feat: update media identities on indexing and add a backfill command`.

### Task 4: Media of a page

Repository: core.

**Files:**
- Create: `Classes/Domain/Model/MediaItem.php`, `Classes/Domain/Repository/PageMediaRepository.php`
- Test: `Tests/Functional/Domain/PageMediaRepositoryTest.php`, `Tests/Functional/Domain/Fixtures/PageMedia.csv`

**Interfaces:**
- Produces: `final readonly class MediaItem` with `int $fileUid`, `int $metadataUid`, `string $name`, `string $mimeType`, `?string $publicUrl`, `?string $mediaUri`, `?string $sha256` (null when stale: `sha256_of !== sha1`), `?string $creator`, `?string $copyright`, `?string $source`, `?string $license`, `?string $digitalSourceType`. Empty strings become null.
- Produces: `PageMediaRepository::findByPage(int $pageUid, int $languageId): list<MediaItem>` — references with `tablenames = 'tt_content'` and `fieldname` any, of content elements with `pid = $pageUid` and `sys_language_uid IN ($languageId, -1)`; frontend enable fields on `tt_content` and `sys_file_reference` (`FrontendRestrictionContainer`), live workspace; `sys_file.missing = 0`; only `MediaIdentity::isMedia()`; ordered by `tt_content.colPos`, `tt_content.sorting`, `sys_file_reference.sorting_foreign`; each file once (first occurrence). Metadata: the row with `sys_language_uid = $languageId` (`l10n_parent` = default row) if present, else the default row; `license` and `digitalSourceType` always from the default row. At most three queries per call, independent of the number of content elements.

- [ ] **Step 1: Failing tests:**
  - `listsMediaInPageOrder`: two content elements with an image and a video → two items in `colPos`/`sorting` order with all fields.
  - `fileUsedTwiceIsListedOnce` (Review Focus 1).
  - `hiddenContentAndHiddenReferencesAreSkipped` (Review Focus 4).
  - `translatedMetadataIsUsedExceptLicenceAndSourceType` (Review Focus 5): language 1 → translated creator, default licence and type.
  - `pdfAndOnlineMediaAreSkipped`; `staleHashIsNull`; `pageWithoutMediaGivesEmptyList`.
- [ ] **Step 2:** `composer test:functional` → FAIL.
- [ ] **Step 3:** Implement model and repository.
- [ ] **Step 4:** `composer test` → PASS.
- [ ] **Step 5:** Commit `feat: read the media used on a page`.

### Task 5: Media members of the Record

Repository: core.

**Files:**
- Create: `Classes/Frontend/MediaJsonLdBuilder.php`
- Test: `Tests/Unit/Frontend/MediaJsonLdBuilderTest.php`

**Interfaces:**
- Consumes: `MediaItem`.
- Produces: `MediaJsonLdBuilder::build(list<MediaItem> $items, \Closure $absoluteUrl): array` with keys `image`, `video`, `audio` present only when non-empty; `$absoluteUrl` is `fn(string $url): string`. Entry: `@type` (`ImageObject`/`VideoObject`/`AudioObject` by MIME prefix), `@id` ← `mediaUri`, `contentUrl` ← absolute `publicUrl`, `sha256`, `encodingFormat` ← `mimeType`, `creator` ← `['name' => …]`, `copyrightNotice`, `license`, `isBasedOn` ← `source` only if `filter_var(…, FILTER_VALIDATE_URL)` and scheme `http`/`https`, `digitalSourceType` ← IRI prefix + term. Null members are left out, as in `JsonLdBuilder`.

- [ ] **Step 1: Failing tests:** full item → exact array (assert with `assertSame` on the whole entry, e.g. `'digitalSourceType' => 'https://cv.iptc.org/newscodes/digitalsourcetype/digitalCapture'`, `'creator' => ['name' => 'dpa']`); item with only name and MIME → `['@type' => 'ImageObject', 'encodingFormat' => 'image/jpeg']`; `source = 'Archiv'` → no `isBasedOn`; mixed list → separate `image` and `audio` arrays, order kept; empty list → `[]`.
- [ ] **Step 2:** `composer test:unit` → FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** `composer test:unit` → PASS.
- [ ] **Step 5:** Commit `feat: build media members of the provenance record`.

### Task 6: Emit media and keep cached pages current

Repository: core.

**Files:**
- Modify: `Classes/Frontend/JsonLdHeaderData.php`
- Create: `Classes/Media/MediaCacheTags.php`, `Classes/Hook/FlushMediaPages.php`
- Modify: `Classes/EventListener/UpdateMediaIdentity.php`, `ext_localconf.php`
- Test: extend `Tests/Functional/Frontend/JsonLdOutputTest.php` and its `ProvenancePages.csv` (or a new `MediaPages.csv`); create `Tests/Functional/Media/MediaCacheFlushTest.php`

**Interfaces:**
- Consumes: `PageMediaRepository::findByPage()`, `MediaJsonLdBuilder::build()`, `AbsoluteUrlResolver::resolve()`.
- Produces: `MediaCacheTags::tagFor(int $fileUid): string` → `'sys_file_' . $fileUid`; `MediaCacheTags::flush(int $fileUid): void` → `CacheManager::flushCachesInGroupByTags('pages', [tagFor()])`.
- `JsonLdHeaderData::render()` merges the media members into the Record **before** dispatching `BeforeJsonLdEmittedEvent` (so facts and signing see them), using the request's language id, and adds one `CacheTag` per listed file to `frontend.cache.collector`. `cpsConformance` is computed exactly as before.
- `UpdateMediaIdentity`: after `update()` returns true, call `flush()`; after `retire()`, call `flush()`.
- `FlushMediaPages::processDatamap_afterDatabaseOperations()` registered under `processDatamapClass['provenance-media']`: for table `sys_file_metadata`, read its `file` column and call `flush()`.

- [ ] **Step 1: Failing tests:**
  - `JsonLdOutputTest::recordListsImagesOfThePage`: `image[0]` has `@id`, absolute `contentUrl` starting `http://localhost/fileadmin/`, `sha256`, `license`.
  - `JsonLdOutputTest::incompleteMediaKeepLevelTwo`: page with an image without creator/licence → `cpsConformance === 2`, image entry present.
  - `MediaCacheFlushTest::metadataSaveFlushesTaggedPages`: put an entry with tag `sys_file_1` into the `pages` cache, save `sys_file_metadata` of file 1 via DataHandler → entry gone; an entry tagged `sys_file_2` stays.
  - `MediaCacheFlushTest::fileReplaceFlushesTaggedPages` via `replaceFile()`.
- [ ] **Step 2:** `composer test:functional` → FAIL.
- [ ] **Step 3:** Implement.
- [ ] **Step 4:** `composer test` → PASS.
- [ ] **Step 5:** Commit `feat: emit page media in the provenance record and refresh pages on file changes`.

### Task 7: Media hint in the Pages submodule

Repository: core.

**Files:**
- Modify: `Classes/Backend/PageOverview.php`, `Resources/Private/Templates/PagesModule.html`, `Resources/Private/Language/locallang_mod_pages.xlf`
- Create: `Resources/Private/Partials/Overview/Media.html`
- Test: extend `Tests/Functional/Backend/PageOverviewTest.php` and `Fixtures/PageOverview.csv`

**Interfaces:**
- Consumes: `PageMediaRepository::findByPage($pageUid, 0)`.
- Produces: row key `media` => `array{total: int, incomplete: list<array{fileUid: int, metadataUid: int, name: string, lacks: list<'creator'|'license'|'identifier'|'hash'>}>}` (`identifier` when `mediaUri` is null, `hash` when `sha256` is null). The row-shape docblock of `rows()` is updated; `missing` and `level` are unaffected.
- Template: column "Media" showing `total` and, if `incomplete` is non-empty, "n incomplete" with an expandable list; each file links to the `record_edit` route for `sys_file_metadata` `metadataUid` only if the user has `tables_modify` on `sys_file_metadata`; `identifier`/`hash` lacks show the hint "Run vendor/bin/typo3 provenance:backfill-media".

- [ ] **Step 1: Failing tests:** page with two images, one without licence → `media.total === 2`, `incomplete[0].lacks === ['license']`; page without media → `['total' => 0, 'incomplete' => []]`; level of that page unchanged.
- [ ] **Step 2:** `composer test:functional` → FAIL.
- [ ] **Step 3:** Implement data and template.
- [ ] **Step 4:** `composer test` → PASS; open the module in a local instance (`typo3-provenance-demo`) and check the column renders.
- [ ] **Step 5:** Commit `feat: show incomplete media in the Pages submodule`.

### Task 8: Extension documentation

Repository: core.

**Files:** Modify `Documentation/Editors/Index.rst` (where to enter creator, licence, source type), `Documentation/Configuration/Index.rst` (filemetadata dependency, `provenance:backfill-media`), `Documentation/Developers/Index.rst` (media members, cache tags `sys_file_<uid>`), `CHANGELOG.md` (Unreleased: Added).

- [ ] **Step 1:** Write the docs.
- [ ] **Step 2:** If `Build/` has a docs render check, run it; otherwise confirm the `.rst` files have no broken references with `grep -n ':ref:' Documentation -r`.
- [ ] **Step 3:** Commit `docs: describe media provenance`.

### Task 9: Specification

Repository: spec (then mirror in site).

**Files:** Modify `specification.md` (Sections "Changes since Draft 0.4", 3.1, 5, 5.2), `vocabulary/v1/context.jsonld` if needed; site `ns/v1/context.jsonld` and `ns/v1/index.html` if the vocabulary changes.

**Interfaces:**
- 3.1: optional members `image`, `video`, `audio` with the entry table from the design ("Record shape"), not part of completeness; the two `sha256` sentences from the design ("Specification changes") verbatim, including "Publishers MUST re-issue the Record when a listed file is replaced, so the signed hash matches the file at `contentUrl`."; a note that the signature covers the media list and that standalone media records are left to a later draft.
- 5: C2PA row mentions media; 5.2 or a short paragraph names MX Carrier Formats §3.5/§3.7 and MX's IPTC `digitalSourceType` derivation as compatible, not adopted.
- Vocabulary: check the current schema.org release for `sha256` and `digitalSourceType`; only if a term is missing or pending, define it in the CPS context.

- [ ] **Step 1:** Edit the spec; add two rows to "Changes since Draft 0.4".
- [ ] **Step 2:** Validate the JSON-LD context(s) with `python3 -m json.tool`.
- [ ] **Step 3:** Commit in each changed repo `docs: add media members to the provenance record`.

### Task 10: Conformance vector

Repository: conformance.

**Files:** Create `vectors/canonicalization/08-record-media/` with `input.html`, `record.json`, `expected.json`, `expected-text.txt` following `07-record-jcs`; update `README.md` vector list.

- [ ] **Step 1:** Record with one `image` entry (all members); `input.html` with the matching `<img alt>`.
- [ ] **Step 2:** Generate expected bytes with `reference/python` and check the reference test suite passes (`README.md` names the command).
- [ ] **Step 3:** Commit `test: add a record vector with media members`.
