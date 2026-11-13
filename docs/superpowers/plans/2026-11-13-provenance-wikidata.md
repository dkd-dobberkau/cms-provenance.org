# provenance_wikidata Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Editors can look up a Wikidata item and its statements inside the CKEditor "Fact" dialog and fill the fact fields from a statement with one click.

**Architecture:** `provenance_facts` gains a generic lookup extension point (`editor.config.provenanceFact.lookups`, dynamically imported ES modules that receive a `context`). The new extension `provenance_wikidata` registers one lookup module through `AfterPrepareConfigurationForEditorEvent`; its PHP side wraps the Wikidata Action API (cached, with User-Agent) behind two backend AJAX routes that return ready-to-display rows, so the JavaScript only renders and calls `fill()`.

**Tech Stack:** PHP 8.2+, TYPO3 13.4/14 (RequestFactory, caching framework, backend AJAX routes, rte_ckeditor event), CKEditor 5 plugin JS (ES modules via TYPO3 import map), PHPUnit via typo3/testing-framework ^9.7.

**Spec:** `docs/superpowers/specs/2026-11-13-provenance-wikidata-design.md` (in `cms-provenance.org`). Read it before starting.

## Global Constraints

- PHP `^8.2`; TYPO3 `^13.4 || ^14.0`; license GPL-2.0-or-later.
- Extension key `provenance_wikidata`; Composer `cms-provenance/provenance-wikidata`; namespace `CmsProvenance\ProvenanceWikidata`; repository `dkd-dobberkau/typo3-provenance-wikidata`; local clone `~/Versioncontrol/local/typo3-provenance-wikidata`.
- Requires `cms-provenance/provenance-facts: ^0.1@dev` (VCS repository entries like `typo3-provenance-audit/composer.json`). No tags, no TER upload, no Packagist registration.
- Nothing in the frontend render path or signing path may call Wikidata. The browser never calls Wikidata directly.
- `provenance_facts` must not mention Wikidata anywhere (code, labels, docs).
- Wikidata entity URIs: `http://www.wikidata.org/entity/Q…`; property URIs: `http://www.wikidata.org/prop/direct/P…`.
- User-Agent: `provenance-wikidata/0.1 (<contact URL>)`, contact URL from extension setting, default `https://github.com/dkd-dobberkau/typo3-provenance-wikidata`.
- Evidence is the statement's first reference URL (P854) or empty; never the Wikidata item URL.
- Code, comments, UI texts and docs in English. Commit messages end with the attribution lines from the session.
- Repo conventions (README, CHANGELOG, CONTRIBUTING, LICENSE, CI with lint + PHP×TYPO3 matrix + Trivy, `Build/phpunit/*`) are copied from `typo3-provenance-facts` / `typo3-provenance-audit`.

## Review Focus

1. Wikidata unreachable or slow (timeout, HTTP 429/5xx, invalid JSON) → panel shows "Wikidata is not reachable right now." and the dialog still creates facts. Test in Task 5 and Task 7.
2. Dates at coarse or odd precision (century/decade = precision < 9, BCE, `-0550-00-00`) → displayed sensibly; a date below year precision fills "Value" with the displayed text, never a wrong ISO date. Test in Task 3.
3. Statements whose value is `somevalue`/`novalue` (no `datavalue`) or of an unsupported type (globe coordinate, monolingual text) → shown as text or skipped, never a PHP error. Test in Task 6.
4. Search input that is empty, whitespace, 300 characters long, or an invalid Q-ID like `Q12; DROP` → 400 with a message, no outgoing request. Test in Task 7.
5. Item without a label in the backend user's language → English label, else the Q-ID. Test in Task 6.

---

### Task 1: Lookup extension point in provenance_facts

Repository: `typo3-provenance-facts`.

**Files:**
- Modify: `Resources/Public/JavaScript/fact-plugin.js` (`openDialog`, `buildForm`)
- Modify: `README.md` (one paragraph "Lookups")

**Interfaces:**
- Produces: editor config key `provenanceFact.lookups: string[]` (ES module specifiers). Each module's default export is `async function (context): void` with
  `context = { form: HTMLFormElement, quote: string, panel: HTMLElement, fill(values: {subject?, predicate?, objectLiteral?, objectUri?, sourceUrl?}): void }`.
  `panel` is an empty `div.provenance-fact-lookups` inserted after the quote; `fill` sets the named inputs and adds the class `is-valid` to each changed input.

- [ ] **Step 1: Verify the import map assumption.** In the running demo (`~/Versioncontrol/local/typo3-provenance-demo`, `ddev start`), open a content element in the backend and check in the browser console that `import('@cms-provenance/provenance-facts/fact-plugin.js')` resolves. Expected: resolves (FormEngine renders via ModuleTemplate, which calls `includeAllImports()`). If it does not, stop and report.
- [ ] **Step 2: Implement.** In `openDialog`, read `this.editor.config.get('provenanceFact.lookups') ?? []`; after building the form, for each specifier `await import(specifier)` inside try/catch (`console.warn` on failure) and call its default export with `context`. The modal opens immediately; lookups attach asynchronously.
- [ ] **Step 3: Manual check without lookups.** Demo backend: mark words, press "Fact", create a fact. Expected: dialog unchanged, fact created, no console errors.
- [ ] **Step 4: Lookup path.** Verified with the real lookup in Task 2 Step 4 (placeholder module is imported: add a `console.debug('provenance lookup loaded')` there and check the console when the dialog opens) and in Task 8.
- [ ] **Step 5: README paragraph** describing the contract above (no mention of Wikidata).
- [ ] **Step 6: Commit and push** `feat: let other extensions add lookups to the fact dialog`.

### Task 2: Repository skeleton and editor registration

Repository: new `typo3-provenance-wikidata`.

**Files:**
- Create: `composer.json`, `ext_emconf.php`, `ext_conf_template.txt`, `LICENSE`, `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `.gitignore`, `.github/workflows/ci.yml`, `Build/phpunit/{UnitTests.xml,UnitTestsBootstrap.php,FunctionalTests.xml,FunctionalTestsBootstrap.php}` (copied from facts, adapted)
- Create: `Configuration/Services.yaml`, `Configuration/JavaScriptModules.php`, `Configuration/Caching.php` or `ext_localconf.php` cache registration
- Create: `Classes/EventListener/RegisterLookup.php`
- Create: `Resources/Public/JavaScript/wikidata-lookup.js` (placeholder: default export that does nothing)
- Test: `Tests/Unit/EventListener/RegisterLookupTest.php`

**Interfaces:**
- Produces: JS specifier `@cms-provenance/provenance-wikidata/wikidata-lookup.js`; cache identifier `provenance_wikidata`; extension settings `contactUrl` (string), `cacheLifetime` (int seconds, default 86400), `timeout` (float seconds, default 5).

- [ ] **Step 1: Write the failing test** `RegisterLookupTest::addsModuleToEditorConfiguration`: given configuration `[]`, after `__invoke(event)` `getConfiguration()['provenanceFact']['lookups']` equals `['@cms-provenance/provenance-wikidata/wikidata-lookup.js']`; second test `keepsExistingLookups`: existing `['other.js']` → `['other.js', '@cms-provenance/provenance-wikidata/wikidata-lookup.js']`, and no duplicate when invoked twice.
- [ ] **Step 2: Run** `composer install && composer test:unit` → FAIL (class missing).
- [ ] **Step 3: Implement** `#[AsEventListener] final class RegisterLookup { public function __invoke(AfterPrepareConfigurationForEditorEvent $event): void }`.
- [ ] **Step 4: Run** `composer test:unit` → PASS. Also `composer validate --no-check-lock --no-check-publish`. Then install the extension in the demo (Task 9 Step 1, from the local path or after Step 5's push) and confirm the placeholder's `console.debug` appears when the Fact dialog opens.
- [ ] **Step 5: Create the GitHub repository** `gh repo create dkd-dobberkau/typo3-provenance-wikidata --public --description "Optional TYPO3 extension: research Content Provenance facts in Wikidata from the fact dialog"`, commit `feat: add extension skeleton that registers the Wikidata lookup`, push, wait for CI green.

### Task 3: Wikidata time values

**Files:**
- Create: `Classes/Value/WikidataTime.php`
- Test: `Tests/Unit/Value/WikidataTimeTest.php`

**Interfaces:**
- Produces: `final readonly class WikidataTime { public static function fromValue(array $value): self; public function display(): string; public function factValue(): string; }` where `$value` is the Wikidata `datavalue.value` (`time`, `precision`).

- [ ] **Step 1: Write failing tests** (data provider, `[time, precision] → [display, factValue]`):
  - `+1683-01-01T00:00:00Z`, 9 → `1683`, `1683`
  - `+1793-08-10T00:00:00Z`, 11 → `10 August 1793`, `1793-08-10`
  - `+1793-08-00T00:00:00Z`, 10 → `August 1793`, `1793-08`
  - `-0530-01-01T00:00:00Z`, 9 → `530 BCE`, `530 BCE`
  - `-0550-00-00T00:00:00Z`, 9 → `550 BCE`, `550 BCE`
  - `+1700-00-00T00:00:00Z`, 7 → `18th century`, `18th century`
  - `+1650-00-00T00:00:00Z`, 8 → `1650s`, `1650s`
- [ ] **Step 2: Run** `composer test:unit` → FAIL.
- [ ] **Step 3: Implement.** Century from year: `intdiv(abs(year) - 1, 100) + 1` with English ordinal suffix; decade: `intdiv(year, 10) * 10 . 's'`; precision < 7 → display the year as given. `factValue()` equals `display()` below precision 9. These literals are what `check_wikidata.py` (`_parse_date`) parses for years, months, days and BCE.
- [ ] **Step 4: Run** → PASS. **Step 5: Commit** `feat: format Wikidata dates at their precision`.

### Task 4: Property mapping

**Files:**
- Create: `Classes/Value/PropertyMapping.php`
- Test: `Tests/Unit/Value/PropertyMappingTest.php`

**Interfaces:**
- Produces: `final class PropertyMapping { public static function predicateFor(string $propertyId): string; }` returning the schema.org term or `http://www.wikidata.org/prop/direct/P…`.

- [ ] **Step 1: Failing tests:** P571→`foundingDate`, P577→`datePublished`, P569→`birthDate`, P570→`deathDate`, P112→`founder`, P50→`author`, P131→`containedInPlace`, P276→`location`, P22→`parent`, P25→`parent`, P1082→`http://www.wikidata.org/prop/direct/P1082`.
- [ ] **Step 2–4:** Run → FAIL; implement as a constant map (reverse of `PROPERTIES` in `content-provenance-conformance/reference/python/check_wikidata.py`; P131→`containedInPlace`, P276→`location` so both round-trip); run → PASS.
- [ ] **Step 5: Commit** `feat: map Wikidata properties to schema.org terms`.

### Task 5: Wikidata client

**Files:**
- Create: `Classes/Wikidata/WikidataClient.php`, `Classes/Wikidata/WikidataUnavailable.php` (exception)
- Test: `Tests/Unit/Wikidata/WikidataClientTest.php`, fixtures `Tests/Unit/Fixtures/search-ashmolean.json`, `Tests/Unit/Fixtures/entities-Q636400.json`, `Tests/Unit/Fixtures/labels.json` (recorded once from the live API, trimmed)

**Interfaces:**
- Consumes: `RequestFactory`, `FrontendInterface` cache `provenance_wikidata`, `ExtensionConfiguration`.
- Produces:
  - `searchItems(string $query, string $language): list<array{id: string, label: string, description: string}>` (`wbsearchentities`, `type=item`, `limit=10`, `uselang=$language`)
  - `getEntities(list<string> $ids, string $language): array<string, array>` (`wbgetentities`, `props=labels|descriptions|claims`, `languages=$language|en`, at most 50 ids per request, chunked)
  - `getLabels(list<string> $ids, string $language): array<string, string>` (labels only, `$language` then `en` then the id)
  - throws `WikidataUnavailable` on timeout, non-200, or invalid JSON.

- [ ] **Step 1: Failing tests** with a stubbed `RequestFactory` returning fixture responses: search returns `Q636400` first with label `Ashmolean Museum`; request carries header `User-Agent: provenance-wikidata/0.1 (https://github.com/dkd-dobberkau/typo3-provenance-wikidata)` and option `timeout` = 5.0; second identical call hits the cache (stub called once); 120 ids → 3 requests; HTTP 503 → `WikidataUnavailable`; body `not json` → `WikidataUnavailable`; label missing in `de` → falls back to `en`.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS.
- [ ] **Step 5: Commit** `feat: add cached Wikidata API client`.

### Task 6: Statement rows

**Files:**
- Create: `Classes/Wikidata/StatementPresenter.php`
- Test: `Tests/Unit/Wikidata/StatementPresenterTest.php` (uses `entities-Q636400.json` plus a hand-written fixture `entities-edge-cases.json`)

**Interfaces:**
- Consumes: `WikidataClient::getEntities`, `getLabels`; `WikidataTime`; `PropertyMapping`.
- Produces: `rows(string $itemId, string $language): array{item: array{id, label, description, uri}, rows: list<array{property: string, propertyLabel: string, value: string, reference: string, fill: array{predicate: string, objectLiteral: string, objectUri: string, sourceUrl: string}}>}`.

- [ ] **Step 1: Failing tests:**
  - Q636400 contains a row `property=P571`, `value=1683`, `fill.predicate=foundingDate`, `fill.objectLiteral=1683`, `fill.objectUri=''`.
  - An item-valued row (P112 → Q471406) has `value` = the item's label, `fill.objectUri=http://www.wikidata.org/entity/Q471406`, `fill.objectLiteral` = the label.
  - A statement with a P854 reference has `reference` and `fill.sourceUrl` set to that URL; without one both are `''`.
  - Deprecated statements are absent; preferred statements come before normal ones of the same property.
  - Edge cases fixture: `somevalue`/`novalue` snaks and globe-coordinate values produce no PHP error (skipped); quantity `+1000000` with unit `1` displays `1000000`; string values display as is.
  - Missing label in `de` → English; missing in both → Q-ID.
  - Labels of all referenced items and properties are fetched with a single `getLabels` call.
- [ ] **Step 2–4:** Run → FAIL; implement; run → PASS.
- [ ] **Step 5: Commit** `feat: present Wikidata statements as fact suggestions`.

### Task 7: Backend AJAX routes

**Files:**
- Create: `Configuration/Backend/AjaxRoutes.php`, `Classes/Backend/LookupController.php`
- Test: `Tests/Functional/LookupControllerTest.php`, `Tests/Functional/Fixtures/BackendUsers.csv`

**Interfaces:**
- Consumes: `WikidataClient::searchItems`, `StatementPresenter::rows`.
- Produces: routes `provenance_wikidata_search` (`GET /provenance-wikidata/search?query=…`) → `{items: [...]}` and `provenance_wikidata_statements` (`GET /provenance-wikidata/statements?item=Q…`) → the `rows()` structure. Language: backend user's `lang`, `default` → `en`.

- [ ] **Step 1: Failing functional tests** (client and presenter replaced by test doubles in the container):
  - admin user, `query=Ashmolean` → 200 with items.
  - user without `tables_modify` on `tt_content` → 403 `{error: …}`.
  - `query=''`, `query='   '`, 300-character query → 400; no client call.
  - `item=Q12; DROP` → 400; `item=Q636400` → 200.
  - client throws `WikidataUnavailable` → 503 `{error: "Wikidata is not reachable right now."}`.
- [ ] **Step 2–4:** Run `composer test:functional` → FAIL; implement (query max 200 chars, item `/^Q[1-9]\d{0,11}$/`); run → PASS.
- [ ] **Step 5: Commit** `feat: add backend routes for Wikidata search and statements`.

### Task 8: Research panel

**Files:**
- Modify: `Resources/Public/JavaScript/wikidata-lookup.js`
- Create: `Resources/Public/Css/wikidata-lookup.css` only if Bootstrap classes are not enough (load via `<link>` added by the module).

**Interfaces:**
- Consumes: Task 1 `context`; Task 7 routes via `TYPO3.settings.ajaxUrls.provenance_wikidata_search` / `_statements` and `AjaxRequest`.

- [ ] **Step 1: Implement** in `context.panel`: label "Look up in Wikidata", search input (debounce 300 ms, minimum 2 characters), result list (label, Q-ID, description). Picking an item calls `fill({ subject: item.uri })` and loads statements as a table (property label, value, reference link, "Use" button). "Use" calls `fill(row.fill)` with all four keys, so a previous object URI is cleared. Errors show the server's `error` text in an `alert alert-warning` inside the panel. Up to 5 values per property, then "Show all".
- [ ] **Step 2: Manual acceptance in the demo** (Task 9 installs it; do Task 9 Step 1 first if needed): search "Ashmolean", pick Q636400, use "inception 1683" → predicate `foundingDate`, value `1683`, subject `http://www.wikidata.org/entity/Q636400`. Use "founded by" → value-as-URI filled, value = label. To simulate an outage, set the extension setting `timeout` to `0.001` and flush the cache: the warning appears in the panel and "Create fact" still works. Reset `timeout` afterwards.
- [ ] **Step 3: Commit and push** `feat: add Wikidata research panel to the fact dialog`; CI green.

### Task 9: Demo, documentation, PRD

**Files:**
- Modify: `typo3-provenance-demo/composer.json` (+ `composer.lock`): require `cms-provenance/provenance-wikidata: @dev`, repository entry `{"type": "git", "url": "https://github.com/dkd-dobberkau/typo3-provenance-wikidata.git"}`
- Modify: `typo3-provenance/Documentation/Editors/Index.rst` (section "Researching a fact in Wikidata"), `typo3-provenance/Documentation/Developers/Index.rst` (section "Adding a lookup to the fact dialog", the Task 1 contract)
- Modify: `typo3-provenance-wikidata/README.md`, `CHANGELOG.md` (Unreleased)
- Modify: `cms-provenance.org/docs/prd-typo3-extensions.md` §3: one paragraph listing `provenance_wikidata` as optional editorial aid outside the conformance levels

- [ ] **Step 1: Demo** `ddev composer require cms-provenance/provenance-wikidata:@dev`, `ddev exec vendor/bin/typo3 extension:setup`, `ddev exec vendor/bin/typo3 cache:flush`.
- [ ] **Step 2: Acceptance** on the local demo: create one fact via the panel on the Ashmolean element; then run in `content-provenance-conformance/reference/python`: `check_facts.py`, `verify_site.py`, `check_site.py` (expected: 0 findings, all valid, `reaches L4 Citable (claims L4)`) and `check_wikidata.py` (expected: the new fact `confirmed`, 0 contradicted). Do not commit that fact; cold-start the demo afterwards (`ddev delete -Oy`, `rm config/system/settings.php`, `ddev start`).
- [ ] **Step 3: Demo CI** stays green after pushing the composer change (`gh run watch`).
- [ ] **Step 4: Docs** written; render check of the core manual if the repo has a render command, else RST lint by eye.
- [ ] **Step 5: Commit and push** each repository; update `cms-provenance.org` build log only if the user asks.
