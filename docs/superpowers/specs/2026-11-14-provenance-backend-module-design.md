# Design: "Provenance" backend module with a facts overview

Date: 2026-11-14. Status: draft for approval.

## Intent

Editors and site administrators have no overview of provenance in the TYPO3 backend. Facts are inline records of single content elements; finding all facts of a site, or seeing which pages lack what, means opening record after record. The user wants an own backend module, first of all with an overview of the facts ("eine Datenbank mit den Facts" in the sense of a browsable, searchable list).

Later, an AI should suggest further facts per page ("context management"). That is not part of this work, but the module is structured so a suggestion feature can plug in.

## Goals

- A backend module "Provenance" in the Web/Content area, following the page tree like the Info module.
- Submodule "Facts" (from `provenance_facts`): all facts of the selected page and its subpages, searchable and filterable, each linking to its content element for editing.
- Submodule "Pages" (from the core `provenance`): per page in the subtree, whether it has a record, the level its record reaches without signing (L1/L2) and what is missing for the next level.
- TYPO3 13.4 and 14, PHP 8.2+, like the other extensions.

## Non-goals (this version)

- AI fact suggestions. Only the extension point that would carry them is considered (see "Later").
- A reusable fact library (one fact used on many pages). Facts stay bound to their quote in one content element.
- Editing facts inside the module. Editing stays in the regular record form; the module links there.
- Export (CSV, JSON). The public representation of facts is the Web Annotation endpoint.
- Signing status and the L3/L4 level in the Pages submodule. The signed level is computed at render time by the signing extension; showing it needs a contribution from that extension (see "Later").
- Wikidata status per fact. It needs the scheduled check that is itself a later extension of `provenance_wikidata`.
- A conformance badge in the page tree (PRD story E-1). The module is the chosen form; the badge may follow.

## Anti-goals

- The module must not show editors pages or facts they cannot see in the page tree (page permissions, mounts) or reveal facts of other workspaces.
- No new public endpoint and no change to what pages publish. The module is backend-only and read-only.
- `provenance_facts` must keep working without the module being opened; the core must not depend on `provenance_facts`.

## Constraints

- Module registration via `Configuration/Backend/Modules.php`. Parent of the main module: `web` (TYPO3 14 renamed it to `content` and keeps `web` as alias), so one configuration serves 13.4 and 14.
- Main module modelled on the core "Status/Info" module (`content_status`, alias `web_info`): `navigationComponent` page tree, `appearance.dependsOnSubmodules`, `showSubmoduleOverview`. Submodules use `parent: provenance`, so they appear in the module's submodule menu.
- Access: `user` (editors), with the regular page permission checks; no admin-only parts.
- Code, labels and docs in English with XLIFF labels, like the rest of the extensions.
- Tests: functional tests for the data queries and permissions; controller responses tested through the module's PSR-7 request handling where practical.

## Approaches considered

- **A. Main module in the core, submodules per extension** (chosen). The core registers "Provenance" and its "Pages" submodule; `provenance_facts` registers the "Facts" submodule. Each extension shows its own data; a site without facts still gets the pages overview; a later AI extension adds its own submodule or contributes to "Facts".
- **B. One module in `provenance_facts` only.** Quicker, but the pages overview (core data) would depend on the facts extension.
- **C. A new extension `provenance_backend`.** Another repository and dependency without a gain.

Chosen by the user: A.

## Design

### Main module (core `provenance`)

- Identifier `provenance`, parent `web`, positioned after the Info/Status module, label "Provenance", own module icon.
- No content of its own: with `showSubmoduleOverview` it lists its submodules (TYPO3 standard behaviour).

### Submodule "Pages" (core)

- Identifier `provenance_pages`, parent `provenance`, page tree navigation.
- Shows the selected page and its subpages up to a depth chosen in the module header (0, 1, 2, 3, 4, all; default 1 like Info modules), only standard pages (doktype 1) the user may see.
- Columns: page title (link to the page properties tab "Provenance"), record (yes/no), level from the record without signing (L1 or L2, computed by `ConformanceLevel`), what is missing for the next level, first published, last substantive revision, last confirmed as still accurate.
- `ConformanceLevel` gains a method that returns the reasons a record does not reach L2 (missing required members, author without name or profile URI, source without description or access date), in the same terms the specification uses. The existing `of()` and `isComplete()` keep their behaviour and share the logic.
- A note above the table says that L3/L4 depend on signing and are visible in the page source or with the conformance tools.

### Submodule "Facts" (`provenance_facts`)

- Identifier `provenance_facts`, parent `provenance`, page tree navigation.
- Data: facts (`tx_provenancefacts_fact`) whose content element lies on the selected page or a subpage up to the chosen depth (same depth control as "Pages"), only pages the user may see, live data (see "Decisions settled after review"). Hidden facts, facts of hidden content elements and facts on hidden pages are shown and marked as such, because editors need to find them.
- Columns: fact ID, quote, subject, predicate, value (literal or URI), evidence (host, full URL as link), confidence, valid from/until, page (title, link to the page in the Page module), and an edit action opening the content element in the record form.
  - Subject and value URIs are shown shortened (Wikidata entity URIs as their Q-ID, other URIs as host and path) with the full URI as title and link.
  - Predicates are shown as written; bare schema.org terms stay bare.
- Search: one text field matching fact ID, quote, subject, value and evidence (case-insensitive, contains). Filters: predicate (the distinct predicates of the current result set) and confidence.
- Sorting by page, then by the content element's position, then by the fact's sorting; pagination with 50 rows per page (TYPO3 core pagination).
- Empty state: a short text explaining how facts are created (Fact button in the rich text editor) with a link to the manual chapter.

### Shared pieces

- Depth control and the subtree query are the same in both submodules. The core provides the subtree query (page ids of a page and its subpages up to a depth, filtered by the user's page permissions) as a small public service, and `provenance_facts` uses it.
- Templates use the TYPO3 backend Fluid layout (`ModuleTemplate`), Bootstrap tables, and the standard doc header (page path, shortcut button, submodule menu).

### Later (not built, but planned for)

- **AI fact suggestions ("context management")**: a separate optional extension adds a submodule or an action per page that asks a model for candidate facts from the page text, stored as suggestions (not as facts), which an editor accepts into the regular Fact form. The "Facts" submodule dispatches a PSR-14 event while building each row and the page header, so such an extension can add an action or a column without changing `provenance_facts`.
- **Signing column** in "Pages": `provenance_signing` contributes "signed at" and the signed level through a PSR-14 event dispatched by the core while building each page row.
- **Wikidata status** in "Facts": `provenance_wikidata` contributes a column once its scheduled check exists.

### Tests

- Core: subtree service (depth, permissions, doktypes); `ConformanceLevel` reasons (unit); "Pages" controller renders levels and reasons for a fixture tree and hides pages without read permission.
- Facts: query (subtree, workspace overlay, hidden states, search, filters, pagination) as functional tests with CSV fixtures; controller renders rows and the edit link; a non-admin editor without access to a subtree sees none of its facts.
- Manual check in the demo on TYPO3 14, and one run on a 13.4 instance or the CI matrix.

### Documentation

- Manual: chapter "For editors" gains a section on the module; "Developers" documents the PSR-14 events and the subtree service; "Known problems" drops "No conformance badge in the page tree" only if the badge is built (it is not, so it stays, with a pointer to the module).
- CHANGELOG entries in both repositories; PRD §8 notes the module as the chosen backend surface.

## Decisions settled after review

Calls made by Claude, open to correction:

- **Which record "Pages" rates:** the stored page record as `JsonLdBuilder` builds it, without dispatching `BeforeJsonLdEmittedEvent` (it needs a frontend request). The column is labelled as the level of the stored record; a note says listeners may change what the page publishes. No change to the core's event contract.
- **Workspaces:** both submodules show live data in this version. When the user works in a workspace, a notice says so. Workspace previews follow later if asked for. (This replaces "in the user's current workspace" in the Facts section above.)
- **Languages:** default language only (`sys_language_uid = 0`) in both submodules, stated in the module. Translated pages and elements and their facts follow later with a language column.
- **Content element records:** "Pages" rates the page-level record only; records of single content elements are not listed or aggregated.
- **Hidden things in "Pages":** hidden pages and hidden records are listed and marked, as in "Facts", instead of appearing as "no record".
- **Permissions:** rows require page read permission (mounts, page permissions) and `tables_select` for the shown table (`pages`, `tx_provenancefacts_fact`). The edit action appears only when the user may modify `tt_content` and edit content on that page; otherwise the row is read-only.

## Left to the builder

- Icons, exact labels, column widths, the position of the module among its siblings.
- Whether filters submit as GET parameters or via module data (`moduleData`) persistence; persisting the last depth and filters per user is welcome.
- Event names and payload shape, as long as the "Later" items can be built on them without changing the extensions again.
