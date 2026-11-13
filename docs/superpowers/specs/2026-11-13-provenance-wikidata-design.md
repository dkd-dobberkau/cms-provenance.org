# Design: provenance_wikidata, Wikidata research in the Fact dialog

Date: 2026-11-13. Status: draft for approval.

## Intent

Editors mark a few words in the rich text editor as a Fact (CKEditor button "Fact", extension `provenance_facts`). Today they type subject, predicate, value and evidence by hand, usually as free text or Wikipedia URLs. The new optional extension `provenance_wikidata` helps them research the fact at exactly that moment: find the Wikidata item the fact is about, see what Wikidata states about it and which references it gives, and fill the dialog from a statement with one click.

It is for everyday editorial work on production sites, not a showcase. A side effect is that new facts carry stable, language-neutral Wikidata identifiers, which makes them checkable later (`check_wikidata.py` in the conformance repository already does that from the outside).

## Goals

- In the Fact dialog, search Wikidata items by name and pick one as the subject.
- Show the picked item's statements in readable form (property label, value, references), in the backend user's language with English as fallback.
- Picking a statement fills predicate, value or value as URI, and evidence.
- Works for TYPO3 13.4 and 14, like the other provenance extensions.

## Non-goals (first version)

- No scheduled check of existing facts and no dashboard widget. That is a later extension of the same package and the reason the Wikidata client is a separate PHP service.
- No SPARQL. Search and item lookup use the Wikidata Action API (`wbsearchentities`, `wbgetentities`); SPARQL is only worth it for the later batch check or queries across many items.
- No research in the inline fact form of the content element (TCA). Research happens where facts are made: the CKEditor dialog.
- No other sources (GND, ORCID). The extension point allows them; none is built.

## Anti-goals

- Nothing in the frontend render path or the signing path ever calls Wikidata. Wikidata being slow or down must not affect page delivery, signing or the audit endpoint.
- Nothing from Wikidata is written into the signed record beyond what the editor accepts into the dialog fields. There is no "confirmed by Wikidata" flag in the record; that would be a specification decision.
- The browser never calls Wikidata directly. All requests go through TYPO3 (backend CSP, Wikimedia User-Agent policy, caching).
- `provenance_facts` must not depend on or mention Wikidata.

## Constraints

- PHP 8.2 minimum (like core and facts), TYPO3 `^13.4 || ^14.0`, GPL-2.0-or-later. Extension key `provenance_wikidata` is registered in TER; Composer name `cms-provenance/provenance-wikidata`; namespace `CmsProvenance\ProvenanceWikidata`.
- Repository `typo3-provenance-wikidata` under the same account as the others, with README, CHANGELOG, CONTRIBUTING, LICENSE, Documentation link to the core manual, and CI copied from `typo3-provenance-audit` (lint, PHP × TYPO3 test matrix, Trivy).
- Wikimedia API etiquette: a descriptive User-Agent with a contact URL, short timeouts, cached responses, no request per keystroke without debounce.
- Code, comments and docs in English; the dialog's existing texts are English-only, and the new texts follow it.

## Approaches considered

- **A. Extension point in `provenance_facts`, Wikidata in its own extension** (chosen). The Fact dialog loads "lookup" modules registered by other extensions; `provenance_wikidata` registers one. Research happens in the dialog where the fact is created; facts stays free of Wikidata, and other sources can plug in later. Costs changes in two repositories.
- **B. Wikidata directly in `provenance_facts`**, switchable by an extension setting. Fastest to build, one repository, but puts an external service into an extension every L4 site installs, contrary to the earlier decision to keep it separate.
- **C. A second CKEditor button in `provenance_wikidata`** with its own dialog that then opens the Fact dialog prefilled. Almost no change in facts, but two buttons and two dialogs for one task; editors have to know which to press.

Chosen by the user: A.

## Design

### Extension point in provenance_facts

- The editor configuration can carry a list of lookup modules: `editor.config.provenanceFact.lookups` (list of ES module specifiers). `provenance_wikidata` adds its module there through a listener for the rte_ckeditor event `AfterPrepareConfigurationForEditorEvent`, so no RTE preset has to be edited and the button keeps working without the extension.
- `fact-plugin.js` dynamically imports each listed module when the dialog opens. A lookup module exports a default function `(context) => void` that may add UI to the form. `context` gives it: the form element, the selected quote, and `fill(values)` which sets any of `subject`, `predicate`, `objectLiteral`, `objectUri`, `sourceUrl` and marks the filled fields visually.
- A module that fails to load is skipped with a console warning; the dialog works as before.
- The contract is documented in the core manual (chapter for developers) so other lookups can be written.

### The research panel (provenance_wikidata, JavaScript)

Placed at the top of the dialog, below the quote:

```
┌ New fact ────────────────────────────────────────────┐
│ "opened in 1683"                                      │
│ Look up in Wikidata  [ Ashmolean            ] (Search)│
│   ○ Ashmolean Museum (Q636400) – museum in Oxford     │
│   ○ Ashmolean Building (Q…)    – …                    │
│ Statements of Ashmolean Museum (Q636400)              │
│   inception        1683            ref: (url)  [Use]  │
│   founded by       Elias Ashmole               [Use]  │
│   located in       Oxford                      [Use]  │
│ ─────────────────────────────────────────────────────│
│ Fact ID / Subject / Predicate / Value / … (as today)  │
└──────────────────────────────────────────────────────┘
```

- Search: debounced input, results with label, Q-ID and description. Picking an item sets the subject to `http://www.wikidata.org/entity/Q…` and loads its statements.
- Statements: one row per property and value, deprecated ranks hidden, preferred rank first. Values shown as text: item labels, dates at their precision ("1683", "August 1793", "c. 530 BCE"), quantities with unit, strings and URLs as they are. Long lists are limited per property with "show all".
- "Use" fills:
  - predicate: the schema.org term when the property has one in the mapping (for example P571 → `foundingDate`), otherwise the property URI `http://www.wikidata.org/prop/direct/P…`;
  - value: item values go to "Value as URI" (`…/entity/Q…`) and their label to "Value"; dates go to "Value" as ISO at their precision (`1683`, `1793-08`, `1793-08-10`, `530 BCE`); other values as text;
  - evidence: the statement's first reference URL (P854) if present, else left empty. The Wikidata item itself is not used as evidence.
- The selected quote stays in the "Value" field until a statement is used; the editor can always edit every field afterwards.

### Server side (provenance_wikidata, PHP)

- `WikidataClient` service: wraps the Action API via TYPO3's `RequestFactory`, sets the User-Agent, a timeout of a few seconds, and caches responses in a dedicated cache (TYPO3 caching framework, database backend by default, lifetime one day, configurable). Reusable later by the scheduled check.
- `StatementPresenter`: turns the raw `wbgetentities` claims into display rows, resolving labels of referenced items and properties in one batched call.
- `PropertyMapping`: the schema.org ↔ Wikidata table, ported from `check_wikidata.py` (`PROPERTIES`) in the reverse direction, kept in one PHP class.
- Two backend AJAX routes, available to backend users who may edit content: `provenance_wikidata_search` (query, language → items) and `provenance_wikidata_statements` (Q-ID, language → rows). Input validated (query length, Q-ID pattern). Errors from Wikidata become a short message in the panel ("Wikidata is not reachable right now"), never a broken dialog.
- Extension settings: User-Agent contact URL (default the repository URL), cache lifetime, timeout.

### Tests

- Unit tests for date formatting at each precision (including BCE), the property mapping, the statement presenter, and request validation, against recorded Wikidata JSON fixtures; no live calls in CI.
- Functional test for the AJAX routes with a mocked client.
- `provenance_facts`: the dialog still works without any lookup registered, and a registered lookup is loaded (manual browser check, see "Decisions settled after review"); `provenance_wikidata`: PHP test that its listener adds the module to the editor configuration.
- Manual acceptance on the demo (`typo3-provenance-demo`): create a fact for the Ashmolean via the panel; `check_facts.py`, `check_site.py` and `check_wikidata.py` stay green and the new fact counts as confirmed.

### Documentation

- README of the new repository; a section in the core manual chapter "For editors" (researching a fact) and in the developer chapter (lookup module contract).
- PRD `docs/prd-typo3-extensions.md`: list the fifth extension as optional and outside the conformance levels.

## Decisions settled after review

Calls made by Claude while writing the design, open to correction:

- **Access to the AJAX routes:** backend users who may modify `tt_content` (`tables_modify`), the same people who can open the Fact dialog. No per-element permission check, since the routes only read public Wikidata.
- **JavaScript tests:** no JS test runner is added to either repository (none exists today). The extension point is covered by a PHP test of the editor configuration listener plus a manual browser check in the demo; the JS is kept thin so most logic lives in tested PHP.
- **Import map:** `provenance_wikidata`'s `JavaScriptModules.php` must make its module available in FormEngine (for example with the `backend.form` tag) so that a module named only in `provenanceFact.lookups` resolves. The builder verifies this in TYPO3 13.4 and 14 before building the panel.
- **Dependencies and release:** `provenance_wikidata` requires `cms-provenance/provenance-facts: ^0.1@dev`, like facts requires the core. Release 0.1.0 stays postponed; this work creates the repository and pushes it, with no tags, TER upload or Packagist registration.
- **Demo:** the demo installs `provenance_wikidata` so the panel can be tried in its backend. No new seeded fact; the acceptance check creates a fact on a local instance and is not committed.

## Left to the builder

- Exact UI markup and styling within TYPO3 backend conventions; debounce time; the per-property limit.
- Cache backend details and identifiers.
- Whether the statement rows are fetched with labels in one or two API calls.
- Wording of messages.
