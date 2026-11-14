# Provenance Backend Module Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A backend module "Provenance" with a "Pages" submodule (core) and a "Facts" submodule (`provenance_facts`) that give editors a read-only overview of records and facts in a page subtree.

**Architecture:** The core registers the main module (parent `web`) and its "Pages" submodule, and offers a subtree service and per-row PSR-14 events. `provenance_facts` registers the "Facts" submodule with its own query repository. Controllers are plain PSR-7 backend controllers rendering Fluid templates through `ModuleTemplate`; data classes are tested functionally, display helpers by unit tests.

**Tech Stack:** PHP 8.2+, TYPO3 13.4/14 backend modules (`Configuration/Backend/Modules.php`, `ModuleTemplateFactory`, `#[AsController]`), Doctrine DBAL QueryBuilder, Fluid, typo3/testing-framework ^9.7.

**Spec:** `docs/superpowers/specs/2026-11-14-provenance-backend-module-design.md` (in `cms-provenance.org`), including "Decisions settled after review", which override earlier sections.

## Global Constraints

- PHP `^8.2`; TYPO3 `^13.4 || ^14.0`; repos `~/Versioncontrol/local/typo3-provenance` (core) and `~/Versioncontrol/local/typo3-provenance-facts`; work on `main`, push after each task, CI must stay green (PHP 8.2/8.4 × TYPO3 13.4/14).
- Main module identifier `provenance`, parent `web` (alias of `content` in TYPO3 14), `access: user`, page tree navigation, `appearance.dependsOnSubmodules: true`, `showSubmoduleOverview: true`.
- Submodules `provenance_pages` (core) and `provenance_facts` (facts), both `parent: provenance`, `access: user`, page tree navigation.
- Read-only: no record changes from the module; editing goes through `record_edit`.
- Live data only (workspace 0) and default language only (`sys_language_uid = 0`); a notice when the user is in a workspace.
- Rows only for pages the user may read (`getPagePermsClause(Permission::PAGE_SHOW)` and mounts) and only with `tables_select` for `pages` / `tx_provenancefacts_fact`. Edit action only with `tables_modify` on `tt_content` and `Permission::CONTENT_EDIT` on the page.
- Hidden pages, hidden records, hidden content elements and hidden facts are listed and marked; deleted ones never.
- Depth choices 0, 1, 2, 3, 4 and "all" (999), default 1; facts list 50 rows per page.
- The core must not depend on `provenance_facts`. English labels in XLIFF; code and docs in English.
- Commit messages end with the session's attribution lines.

## Review Focus

1. Editor with a DB mount on a subtree selects a page outside it (via URL `id`) → no rows, no error. Test in Task 2 and Task 6.
2. Search text containing `%`, `_` or quotes → treated literally, no SQL error. Test in Task 6.
3. Fact whose content element was deleted (orphan fact) → not listed. Test in Task 6.
4. Page with a record whose `tx_provenance_document` is hidden → listed with "record hidden", not "no record". Test in Task 3.
5. Requested result page beyond the last (`?page=99`) → last page shown (or empty list with pagination), no exception. Test in Task 6.

---

### Task 1: Reasons a record misses L2

Repository: core.

**Files:** Modify `Classes/Service/ConformanceLevel.php`; Test `Tests/Unit/Service/ConformanceLevelTest.php` (extend existing).

**Interfaces:**
- Produces: `ConformanceLevel::missing(array $record): list<string>` — English sentences, empty when the record is complete. `isComplete($record)` becomes `missing($record) === []`.

- [ ] **Step 1: Failing tests** (data provider, record → expected list):
  - complete record → `[]`
  - no vocabulary context → `['The record does not use the vocabulary https://cms-provenance.org/ns/v1.']`
  - missing `datePublished` → `['The record has no datePublished.']` (same pattern for each REQUIRED member)
  - no author → `['The record names no author.']`
  - author `{"name": "Ada"}` without `@id` → `['Author "Ada" has no profile URI.']`; author without name and id → `['An author has no profile URI.']`
  - source `{"url": "https://example.org/a", "name": "A"}` → `['Source https://example.org/a has no access date.']`; missing name → `'Source https://example.org/a has no description.'`
  - several problems → all, in the order above.
- [ ] **Step 2:** `composer test:unit` → FAIL.
- [ ] **Step 3:** Implement `missing()`; rewrite `isComplete()` on top of it; existing tests must stay green.
- [ ] **Step 4:** `composer test` → PASS. **Step 5:** commit `feat: explain why a record does not reach L2`.

### Task 2: Subtree service

Repository: core.

**Files:** Create `Classes/Backend/PageSubtree.php`; Test `Tests/Functional/Backend/PageSubtreeTest.php`, fixture `Tests/Functional/Backend/Fixtures/PageTree.csv` (pages incl. hidden, deleted, translated, workspace version, a branch only admins may read; be_users: admin, editor with `db_mountpoints` on one branch and group perms).

**Interfaces:**
- Produces: `final readonly class PageSubtree { public const ALL = 999; public function pageIds(int $rootUid, int $depth, BackendUserAuthentication $user): list<int> }` — root first, then breadth-first; live default-language pages only; hidden included, deleted excluded; filtered by `getPagePermsClause(Permission::PAGE_SHOW)` and the user's mounts (`isInWebMount`); a page the user may not read stops descent below it. Public service (used by facts).

- [ ] **Step 1: Failing tests:** depth 0 → `[root]`; depth 1 → root + children incl. hidden child; deleted child absent; translated and workspace rows absent; depth ALL reaches the deepest page; editor outside mount → `[]` (Review Focus 1); editor inside mount sees only readable pages.
- [ ] **Step 2–4:** run → FAIL; implement with `QueryBuilder` on `pages` (`DeletedRestriction` only, `sys_language_uid = 0`, `t3ver_wsid = 0`, perms clause as `QueryHelper::stripLogicalOperatorPrefix`), one query per depth level; run → PASS.
- [ ] **Step 5:** make the service public in `Configuration/Services.yaml`; commit `feat: add a permission-aware page subtree service`.

### Task 3: Page overview rows

Repository: core.

**Files:** Create `Classes/Backend/PageOverview.php`, `Classes/Event/ModifyPageOverviewRowEvent.php`; Modify `Classes/Domain/Repository/ProvenanceRecordRepository.php` (`findByUid(int $documentUid, bool $includeHidden = false)`); Test `Tests/Functional/Backend/PageOverviewTest.php`, fixture `PageOverview.csv`.

**Interfaces:**
- Consumes: `PageSubtree::pageIds`, `ConformanceLevel::of/missing`, `JsonLdBuilder::build($record, $title, $lastChanged, '')`.
- Produces: `PageOverview::rows(int $rootUid, int $depth, BackendUserAuthentication $user): list<array{uid: int, title: string, pageHidden: bool, hasRecord: bool, recordHidden: bool, level: ?int, missing: list<string>, datePublished: ?string, lastSubstantiveRevision: ?string, maintainedDate: ?string, extra: array<string, string>}>`; standard pages (doktype 1) only. `ModifyPageOverviewRowEvent(array $row, int $pageUid)` with `getRow()/setRow()`, dispatched per row (signing will add to `extra`).

- [ ] **Step 1: Failing tests:** complete record → level 2, `missing` `[]`; incomplete → level 1 with reasons; no record → `hasRecord false`, level null; hidden document → `hasRecord true, recordHidden true` (Review Focus 4); hidden page marked; non-doktype-1 pages skipped; a listener (test fixture extension `Tests/Functional/Fixtures/Extensions/provenance_test_listener`) adding `extra['signed'] = 'yes'` shows up.
- [ ] **Step 2–4:** run → FAIL; implement; run → PASS (whole suite).
- [ ] **Step 5:** commit `feat: build the page overview of the provenance module`.

### Task 4: Main module and "Pages" submodule

Repository: core.

**Files:** Create `Configuration/Backend/Modules.php`, `Classes/Backend/Controller/PagesModuleController.php`, `Resources/Private/Templates/PagesModule.html`, `Resources/Private/Language/locallang_mod.xlf` (main module), `locallang_mod_pages.xlf`, `locallang_module.xlf` (template texts), `Resources/Public/Icons/module-provenance.svg` + icon `module-provenance` in `Configuration/Icons.php`; Modify `Documentation/Editors/Index.rst` (section "The Provenance module"), `Documentation/Developers/Index.rst` (events, `PageSubtree`), `Documentation/KnownProblems/Index.rst` (pointer to the module), `CHANGELOG.md`.

**Interfaces:**
- Consumes: `PageOverview::rows`, `PageSubtree::ALL`.
- Produces: module identifiers `provenance`, `provenance_pages`; route `provenance_pages` (module route); `moduleData` key `depth` (int, default 1) persisted per user; the main module's labels file path `LLL:EXT:provenance/Resources/Private/Language/locallang_mod.xlf` for Task 7.

- [ ] **Step 1:** Controller `#[AsController] final readonly class PagesModuleController { public function handleRequest(ServerRequestInterface $request): ResponseInterface }`: page id from `id` query param; no page selected → info callout "Select a page in the page tree"; depth menu in the doc header (0–4, all); workspace notice when `$user->workspace !== 0`; note that levels are those of the stored record without signing; table per the spec's "Pages" columns; title links to `record_edit` for `pages` (tab Provenance) when editable.
- [ ] **Step 2: Functional smoke test** `Tests/Functional/Backend/PagesModuleControllerTest.php`: request with `module` attribute from `ModuleProvider::getModule('provenance_pages')`, `route`, `normalizedParams`, admin `BE_USER`/`LANG`, `id=1` → 200 and body contains the fixture page title and "L2". If wiring the backend request proves impractical, replace by a manual check and ledger a ruling.
- [ ] **Step 3:** Manual check in the demo (`~/Versioncontrol/local/typo3-provenance-demo`, update `composer.lock` for the core, flush caches): module appears under Web/Content in TYPO3 14; Pages submodule lists the 6 demo pages at L2 (depth "all" on the root); the main module shows its submodule overview.
- [ ] **Step 4:** Docs and CHANGELOG; render docs with `docker run --rm -v "$PWD:/project" ghcr.io/typo3-documentation/render-guides:latest --no-progress --minimal-test --output=/tmp/out Documentation` → no warnings.
- [ ] **Step 5:** `composer test` → PASS; commit `feat: add the Provenance backend module with a pages overview`; push; CI green.

### Task 5: Short labels for URIs

Repository: facts.

**Files:** Create `Classes/Backend/UriLabel.php`; Test `Tests/Unit/Backend/UriLabelTest.php`.

**Interfaces:** Produces `UriLabel::short(string $value): string`.

- [ ] **Step 1: Failing tests:** `http://www.wikidata.org/entity/Q636400` → `Q636400`; `https://www.wikidata.org/wiki/Q42` → `Q42`; `http://www.wikidata.org/prop/direct/P1082` → `P1082`; `https://en.wikipedia.org/wiki/Louvre` → `en.wikipedia.org/wiki/Louvre`; `https://example.org/` → `example.org`; plain text `Daughter of Nabonidus` → unchanged; empty → empty.
- [ ] **Step 2–4:** FAIL → implement → PASS. **Step 5:** commit `feat: shorten URIs for the facts overview`.

### Task 6: Facts overview query

Repository: facts.

**Files:** Create `Classes/Backend/FactOverview.php`, `Classes/Backend/FactFilter.php`, `Classes/Event/ModifyFactOverviewRowEvent.php`; Test `Tests/Functional/Backend/FactOverviewTest.php`, fixture `FactOverview.csv`.

**Interfaces:**
- Consumes: core `PageSubtree::pageIds` (public service).
- Produces:
  - `final readonly class FactFilter { public function __construct(public string $search = '', public string $predicate = '', public string $confidence = '') }`
  - `FactOverview::find(list<int> $pageIds, FactFilter $filter, int $page, int $perPage = 50): array{rows: list<array{uid: int, factId: string, quote: string, subject: string, predicate: string, objectLiteral: string, objectUri: string, sourceUrl: string, confidence: string, validFrom: ?int, validUntil: ?int, factHidden: bool, contentUid: int, contentHidden: bool, pageUid: int, pageTitle: string, pageHidden: bool, extra: array<string, string>}>, total: int, page: int, pages: int}` — `page` clamped to 1..pages.
  - `FactOverview::predicates(list<int> $pageIds): list<string>` (distinct, sorted).
  - `ModifyFactOverviewRowEvent(array $row)` with `getRow()/setRow()`, dispatched per row; `BeforeFactOverviewRenderedEvent` (Task 7) carries header actions.

- [ ] **Step 1: Failing tests:** facts of the given pages only; join to `tt_content` (`content_uid`) and `pages` (`tt_content.pid`), live (`t3ver_wsid = 0`) default-language content only; hidden fact / hidden element / hidden page listed and flagged; deleted fact, deleted element and orphan fact (element missing) absent (Review Focus 3); search matches fact ID, quote, subject, value (literal and URI) and evidence, case-insensitive; search `50%_off"` matches only the literal text (Review Focus 2); predicate and confidence filters; ordering page → element `sorting` → fact `sorting`; 120 facts → 3 pages, `page=99` → page 3 (Review Focus 5); `predicates()` distinct and sorted; empty `$pageIds` → no query, empty result (Review Focus 1).
- [ ] **Step 2–4:** FAIL → implement (`escapeLikeWildcards`, named parameters, `DeletedRestriction` only) → PASS.
- [ ] **Step 5:** commit `feat: query the facts overview`.

### Task 7: "Facts" submodule

Repository: facts.

**Files:** Create `Configuration/Backend/Modules.php`, `Classes/Backend/Controller/FactsModuleController.php`, `Classes/Event/BeforeFactOverviewRenderedEvent.php`, `Resources/Private/Templates/FactsModule.html`, `Resources/Private/Language/locallang_mod_facts.xlf`, `locallang_module.xlf`, module icon; Modify `composer.json` (no new dependency; the core already is one), `README.md`, `CHANGELOG.md`, core manual `Documentation/Editors/Index.rst` (Facts submodule) and `Developers/Index.rst` (both facts events) in the core repo.

**Interfaces:**
- Consumes: `PageSubtree`, `FactOverview::find/predicates`, `FactFilter`, `UriLabel::short`, module `provenance` from Task 4.
- Produces: module `provenance_facts`; `moduleData` keys `depth`, `search`, `predicate`, `confidence`; query param `page`.

- [ ] **Step 1:** Controller like Task 4: no page selected → callout; depth menu; search form (GET, keeps `id`), predicate select (from `predicates()`), confidence select; workspace notice; table per spec columns with `UriLabel::short` for subject/value URIs (full URI as link and title), evidence host as link; badges for hidden fact/element/page; edit action (`record_edit` for `tt_content` uid, `returnUrl` = current module URL) only when permitted; page title links to `web_layout?id=`; pagination (core `SimplePagination` with an `ArrayPaginator` over the current page's rows or own prev/next with `total`/`pages`); empty state text with link to the manual's facts chapter; `BeforeFactOverviewRenderedEvent` dispatched with the selected page id, for header actions.
- [ ] **Step 2:** Functional smoke test `Tests/Functional/Backend/FactsModuleControllerTest.php` as in Task 4: admin, `id=1` → 200, body contains a fixture quote and the shortened `Q636400`; an editor without `tables_select` on `tx_provenancefacts_fact` → body without fact rows; editor without content edit permission → no edit link.
- [ ] **Step 3:** Manual check in the demo (update `composer.lock` for both repos): Facts submodule on the root with depth "all" lists 14 facts; search "1683" finds `ashmolean-opened`; predicate filter `foundingDate` gives 4; edit opens the content element; module renders on TYPO3 14 (and CI covers 13.4 for PHP).
- [ ] **Step 4:** Docs, README, CHANGELOGs; docs render without warnings.
- [ ] **Step 5:** `composer test` in both repos → PASS; commit `feat: add the facts overview to the Provenance module`; push both; CI green.

### Task 8: Demo, PRD

**Files:** Modify `typo3-provenance-demo/composer.lock` (both packages), `typo3-provenance-demo/README.md` ("What to look at": the module), `cms-provenance.org/docs/prd-typo3-extensions.md` §8 (one paragraph: module chosen as backend surface, badge not built).

- [ ] **Step 1:** `ddev composer update cms-provenance/provenance cms-provenance/provenance-facts`; cold start (`ddev delete -Oy`, `rm config/system/settings.php`, `ddev start`); all conformance checks green (`check_endpoint`, `check_facts`, `verify_site`, `check_site`, `check_mx`).
- [ ] **Step 2:** Browser check of both submodules as admin.
- [ ] **Step 3:** Commit and push demo and PRD; demo CI green.
