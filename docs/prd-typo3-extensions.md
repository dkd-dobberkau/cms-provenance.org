# PRD, TYPO3 Content Provenance Extensions

**Status.** Draft 0.1, May 2026. Revised November 2026, see "Revision notes" below.

**Maintainers.** To be assigned. Initial drafting by dkd Internet Service GmbH.

**Related documents.** [Content Provenance Specification](https://github.com/dkd-dobberkau/content-provenance-spec), currently Draft 0.2. Project site: <https://cms-provenance.org>.

**Revision notes, November 2026.**

- Extension keys changed from `content_provenance*` to `provenance*`. The key `content_provenance` was already registered on TER by a third party.
- Vocabulary namespace corrected to `https://cms-provenance.org/ns/v1`, matching the project domain.
- PHP namespace changed from `Typo3\ContentProvenance` to `CmsProvenance\Provenance`. The `TYPO3` vendor namespace is reserved for TYPO3 Core.
- Story S-3 no longer relies on `cruser_id`, which was removed in TYPO3 v12 (Breaking #98024).
- Extension license changed from MIT to GPL-2.0-or-later, the license TYPO3 extensions in TER must use.
- Repositories created (milestone M1): [typo3-provenance](https://github.com/dkd-dobberkau/typo3-provenance), [typo3-provenance-signing](https://github.com/dkd-dobberkau/typo3-provenance-signing), [typo3-provenance-facts](https://github.com/dkd-dobberkau/typo3-provenance-facts), [typo3-provenance-audit](https://github.com/dkd-dobberkau/typo3-provenance-audit), [content-provenance-spec](https://github.com/dkd-dobberkau/content-provenance-spec), [content-provenance-conformance](https://github.com/dkd-dobberkau/content-provenance-conformance). They start under a personal account and move to a neutral organisation once one exists.
- Annex A TCA updated for TYPO3 v13/v14: `required => true` instead of `eval => required`, TCA type `link` instead of `renderType => inputLink` (Feature #97159), associative `items` with `label` and `value`, and no `ctrl.searchFields` (removed in v14, Breaking #106972).

**Audience.** TYPO3 contributors, agency developers, integrators, and anyone who wants to build the reference implementation of the Content Provenance Specification on TYPO3.

This is the product requirements document for the TYPO3 reference implementation of the Content Provenance Specification. It is intentionally written for an open community context. Decisions are explained rather than asserted, alternatives are named, and the boundary between specification, reference implementation, and TYPO3-specific opinion is kept visible throughout.


## 1. Why this exists

The Content Provenance Specification defines four layers that make published content verifiable by automated agents and human auditors. Provenance records, cryptographic signatures, addressable facts, and a well-known audit endpoint. The specification is CMS-agnostic. Without a working reference implementation in at least one CMS, the specification remains theory.

TYPO3 is the right first home for three reasons. The community has a tradition of structured content and clean separation between data and presentation, which the specification rewards. The European customer base, including public sector and regulated industries, will see Cyber Resilience Act adjacent requirements arrive soon, and provenance is an early answer to those. And the TYPO3 Association has the governance maturity to host a specification effort without turning it into a vendor product.

This PRD describes what to build first, how to architect it, and how to keep it open enough that Drupal, WordPress, and Joomla can mirror the approach without rewriting the data model from scratch.

## 2. Goals and non-goals

The extensions described here pursue four concrete goals. To make every TYPO3 page a self-describing document with a complete, machine-readable provenance record. To allow publishers to sign their content with a stable, rotatable key. To make individual claims within a document addressable, not only the document as a whole. And to expose all of this through a well-known endpoint that auditors and AI agents can read.

What these extensions are not. They are not an SEO tool. They do not score, rank, or evaluate content quality. They do not enforce any editorial policy. They do not require a commercial registry. They do not couple to EXT:solr or any other search backend. They do not depend on any TYPO3-specific feature that would prevent the data model from being mirrored in other CMS.

## 3. Architecture, one core plus optional subs

The implementation is split into a small core extension and three optional sub-extensions, each mapping to one of the higher specification layers. This split keeps the entry cost low for adopters who only want provenance records, while letting more ambitious sites add signatures, facts, and audit endpoints incrementally.

**`provenance`.** Core extension. Provides the provenance record data model, the backend forms, the frontend output as JSON-LD, and the public API for other extensions to read and extend records. Implements Layer 1 of the specification.

**`provenance_signing`.** Optional. Adds Ed25519 key management, content hashing, signature generation at publish time, and signature exposure in JSON-LD. Implements Layer 2. Depends on the core.

**`provenance_facts`.** Optional. Adds the fact data model, the backend editor surface for marking up facts within content elements, and the frontend output as Microdata plus Web Annotations. Implements Layer 3. Depends on the core.

**`provenance_audit`.** Optional. Registers the well-known route, generates the paginated JSON listing of all signed documents, exposes the public keys, and serves Web Annotations for facts. Implements Layer 4. Depends on the core and benefits substantially from the signing extension being installed alongside it.

A site that installs only the core reaches Conformance Level L2 Provenance. Adding signing reaches L3 Verifiable. Adding facts and audit reaches L4 Citable. Sites can move up the levels at their own pace.

This split has one downside worth naming. Cross-cutting concerns, especially the canonicalisation algorithm used by signing and by the audit endpoint, must live in the core to avoid duplication. The core therefore ships a canonicalisation service that the signing and audit extensions consume, even though the core itself does not sign. This is a slight violation of single responsibility in favour of avoiding duplication across optional packages, and it is the right tradeoff for an open-source project where forks are likely.

## 4. User stories

Stories are grouped by role. Each story is followed by acceptance criteria.

### Editor

**E-1.** As an editor, I want to see the provenance status of a page in the page module, so I know whether it is publication-ready according to my organisation's standard.

Acceptance criteria. The page module shows a badge per page indicating the current conformance level. Clicking the badge opens a panel listing missing fields. The badge is visible in the page tree and in the page properties form.

**E-2.** As an editor, I want to attach sources to a page or content element, so the provenance record carries verifiable references.

Acceptance criteria. The page properties form and the content element form expose an inline records section for sources. Each source has URL, description, and access date as required fields, plus archive URL as optional. Sources can be reordered, edited, and removed without affecting the document identifier.

**E-3.** As an editor, I want to mark a save as either technical or substantive, so the substantive revision timestamp is meaningful.

Acceptance criteria. The save buttons in the page and content element forms are split into "Save" and "Save as substantive revision". The default is the former. Only the latter updates `last_substantive_revision_at`. The choice is logged in `sys_history`.

**E-4.** As an editor authoring claims, I want to mark a sentence or value as a fact, so it becomes addressable.

Acceptance criteria. The rich text editor offers a fact button. Selecting text and pressing the button opens a dialog where subject, predicate, object, and source are entered. The marked text receives a stable fact identifier. The dialog supports vocabularies declared at the site level, defaulting to schema.org. Editing or removing a fact preserves the surrounding text formatting. This story belongs to `provenance_facts`.

### Author

**A-1.** As an author, I want my external identity, for example an ORCID, to appear on every page I write, so my work is attributable beyond the TYPO3 user database.

Acceptance criteria. The backend user record has fields for external identity URI and verified status. When set, the URI is exposed in the JSON-LD author field of every page where the user is recorded as author. The verified flag is exposed as a separate property and never as part of the URI itself.

### Site administrator

**S-1.** As a site administrator, I want to generate a signing key for my site, so my published content can carry signatures.

Acceptance criteria. A CLI command `provenance:keys:generate` produces an Ed25519 key pair. The private key is written to a path configured in the site configuration. The public key is written both to the configured path and to a publicly accessible URL under `/.well-known/content-provenance/keys/`. The command refuses to overwrite an existing key without an explicit confirmation flag. This story belongs to `provenance_signing`.

**S-2.** As a site administrator, I want to rotate keys, so a compromised key can be replaced without losing the validity of past publications.

Acceptance criteria. A CLI command `provenance:keys:rotate` generates a new key, marks the previous key as retired with a retirement timestamp, and updates the audit endpoint to list both. The previous public key remains accessible at its stable URL. Documents signed under the old key remain valid until republished under the new key or marked as historical. This story belongs to `provenance_signing`.

**S-3.** As a site administrator, I want to backfill provenance records for existing content, so I can adopt the extension without losing my publication history.

Acceptance criteria. A CLI command `provenance:backfill` walks all pages and content elements without a provenance record and creates default records. The first publication timestamp is taken from `crdate`. TYPO3 no longer stores a creating user on the record (`cruser_id` was removed in v12), so the author is taken from the oldest `sys_history` entry for the record where one exists, and left empty for editors to fill otherwise. No sources are added. The command is idempotent and skips records that already exist. A dry-run flag is supported.

### Auditor or AI agent

**X-1.** As an auditor, I want to retrieve a machine-readable list of all signed documents on a site, so I can verify their integrity.

Acceptance criteria. The endpoint at `/.well-known/content-provenance` returns valid JSON conforming to the specification schema. It lists all published documents with URL, identifier, hash, signature, and timestamps. Pagination follows RFC 8288 Link headers. The endpoint responds within reasonable time even on sites with tens of thousands of pages. This story belongs to `provenance_audit`.

**X-2.** As an AI agent, I want to retrieve the public key for a site, so I can verify signatures locally.

Acceptance criteria. The endpoint exposes the public key, the key identifier, the algorithm, and any retirement timestamp. Multiple keys are listed when key rotation has occurred. The format is JWK or PEM at the caller's choice via Accept header. This story belongs to `provenance_audit`.

**X-3.** As an AI agent, I want to retrieve a single fact as a Web Annotation, so I can cite an individual claim rather than an entire page.

Acceptance criteria. The endpoint at `/.well-known/content-provenance/facts/{factId}` returns a valid Web Annotation referencing the source document, the target text fragment, and the structured claim. The annotation conforms to the W3C Web Annotation Data Model. This story belongs to `provenance_facts` plus `provenance_audit`.

## 5. Data model

The data model is described at the conceptual level here. Annex A contains the TCA fragments.

### Tables introduced by the core

**tx_provenance_document.** One record per logical document. Holds the stable identifier, first publication timestamp, substantive revision timestamp, content hash placeholder for signing extensions to fill, reviewer, conformance level, and CMS version. Authors are kept as a comma-separated reference in v0.1 and will move to an MM table in v0.2, named explicitly as deferred for simplicity.

**tx_provenance_source.** Sources attached to a document. URL, description, access date, optional archive URL.

### Tables introduced by the facts extension

**tx_provenance_fact.** Individual facts within a document, each with subject URI, predicate, object literal or URI, optional source reference, optional confidence and validity window.

### Columns added to existing tables

`pages` and `tt_content` each receive a `provenance_document_uid` column referencing `tx_provenance_document`. The relation is one-to-one in v0.1. The fact table additionally references `tt_content` to bind a fact to a specific content element.

`be_users` receives two columns. `external_identity_uri` for ORCID or comparable URIs. `verified` as a boolean indicating that the identity has been confirmed by an admin.

### Why pages and tt_content both, not just pages

A page-level provenance record is sufficient for most sites and matches the typical document boundary. Content-element-level records are useful when a page aggregates content from different authors or different sources, for example a landing page that pulls in expert quotes. Both tables therefore carry the reference, with the page-level record taking precedence when both are present. This is deliberate flexibility and one of the few places where the TYPO3 implementation is more granular than the specification requires.

## 6. API

The core extension exposes a public PHP API plus events. Sub-extensions register against this API rather than against TYPO3 internals where possible, so the same extension structure can be ported to other CMS.

### PHP API

```php
namespace CmsProvenance\Provenance\Api;

interface ProvenanceRecordRepository
{
    public function findByDocument(string $documentUri): ?ProvenanceRecord;
    public function findByPage(int $pageUid): ?ProvenanceRecord;
    public function findByContentElement(int $contentElementUid): ?ProvenanceRecord;
    public function persist(ProvenanceRecord $record): void;
}

interface CanonicalizationService
{
    /**
     * Produce the canonical byte sequence of a document, deterministic
     * across saves, for hashing and signing.
     */
    public function canonicalize(ProvenanceRecord $record, string $renderedHtml): string;
}

interface ProvenanceRenderer
{
    /**
     * Emit the JSON-LD block for a given record. Called by the
     * PageRenderer hook in the core; sub-extensions can decorate
     * the output via events.
     */
    public function renderJsonLd(ProvenanceRecord $record): string;
}
```

### Events

The core dispatches Symfony events that sub-extensions and integrators subscribe to.

`BeforeProvenanceRecordPersisted`. Fired before a record is written to the database. Allows enrichment, for example automatic source detection from RTE content.

`BeforeCanonicalization`. Fired before the canonical byte sequence is computed. Allows registration of custom normalization steps, used by `provenance_signing`.

`BeforeJsonLdEmitted`. Fired before the JSON-LD block is rendered. Allows sub-extensions to add fields, used by signing to add the signature property and by facts to add the claim list.

`AfterDocumentPublished`. Fired after a document has been published, including its provenance record. Used by signing to trigger signature generation asynchronously, and by audit to invalidate the cached endpoint listing.

### CLI commands

The core provides three commands. Signing, facts, and audit register additional commands under their respective namespaces.

```
provenance:document:show <pageUid>
provenance:document:validate <pageUid>
provenance:backfill [--dry-run]
provenance:keys:generate            (from signing)
provenance:keys:rotate              (from signing)
provenance:keys:list                (from signing)
provenance:audit:rebuild-index      (from audit)
provenance:facts:list <pageUid>     (from facts)
```

## 7. Frontend output

The provenance record is emitted as a JSON-LD script in the head of every page that has a record. The block uses the schema.org context plus the provenance extension namespace. Sub-extensions enrich the block via the `BeforeJsonLdEmitted` event rather than rendering separate blocks, so the output remains a single, validatable JSON-LD document.

A minimal example, with only the core installed.

```json
{
  "@context": ["https://schema.org", "https://cms-provenance.org/ns/v1"],
  "@type": "Article",
  "@id": "urn:uuid:7c2f...",
  "headline": "The provenance gap and TYPO3",
  "author": {
    "@type": "Person",
    "@id": "https://orcid.org/0000-0001-2345-6789",
    "name": "Olivier Dobberkau"
  },
  "datePublished": "2026-05-14T09:00:00Z",
  "dateModified": "2026-05-14T09:00:00Z",
  "lastSubstantiveRevision": "2026-05-14T09:00:00Z",
  "citation": [
    {
      "@type": "WebPage",
      "url": "https://mx.allabout.network/blog/the-provenance-gap.html",
      "name": "The provenance gap, Tom Cranstoun",
      "accessDate": "2026-05-14"
    }
  ],
  "conformanceLevel": 2,
  "generator": "TYPO3 14.0 + provenance 0.1"
}
```

With the signing extension installed, the block additionally carries a `signature` object containing the hash, the signature, and the key identifier. With the facts extension, the block carries a `claim` array referencing the facts. The facts themselves are also embedded inline in the body via Microdata attributes, so the visible HTML and the structured data stay in sync without duplication.

## 8. Backend integration

The page module receives a small extension. Each page in the tree shows a tiny badge indicating its conformance level, similar to the existing translation status badges. The page properties form gains a tab labeled "Provenance" containing the document identifier (read-only after first publication), the source list as an inline records section, the author and reviewer fields, and the substantive revision controls. The content element form gains an inline records section for facts when the facts extension is installed.

The RTE integration for facts is the riskiest user experience decision in this PRD. We propose an inline button that opens a dialog rather than a sidebar panel, because a dialog interrupts the flow only at the point of marking up and does not consume horizontal space. We name this as a v0.1 choice that should be revisited after the first round of editor feedback.

## 9. Configuration

Site-level configuration lives in the existing TYPO3 site configuration. The core extension adds the following keys.

```yaml
provenance:
  vocabulary: https://schema.org
  documentIdentifierScheme: uuid           # uuid or url
  defaultAuthorFromBeUser: true
  conformanceLevelTarget: 2

  signing:                                  # only used if signing is installed
    keyPath: /var/lib/typo3/keys/site-key.pem
    keyId: site-key-2026-05
    algorithm: Ed25519
    publicKeyUrl: /.well-known/content-provenance/keys/site-key-2026-05.pem

  audit:                                    # only used if audit is installed
    cacheTtl: 300
    paginationSize: 100
    exposePrivateDocuments: false
```

No global TYPO3-wide configuration is introduced. Sites with multiple site configurations get independent provenance settings, which is the correct behaviour for multi-tenant installations.

## 10. Migration and rollout

Adoption is voluntary and incremental. A site installs the core, sets a configuration target conformance level of 2, runs the backfill command, and is at L2 the same day with no editorial changes. The backfilled records contain only what can be derived automatically. Sources, substantive revisions, and reviewer assignments are added by editors as they touch pages.

Moving from L2 to L3 requires the signing extension and key generation. Existing documents are not signed until they are republished. A command `provenance:document:sign-all` can be run to sign existing documents in bulk, which we recommend doing in a maintenance window because the canonicalisation step is non-trivial for large sites.

Moving from L3 to L4 requires the audit extension and is the simplest step, because the audit endpoint is generated from data already in the database.

Downgrades are supported by uninstalling the relevant sub-extension. The data remains in the database but is no longer exposed. A clean uninstall via `extensionmanager` removes only the sub-extension's tables, not the core's records.

## 11. Performance considerations

Three performance concerns deserve early attention. The canonicalisation step is the most expensive operation in the system, because it walks the rendered HTML and applies normalisation. We mitigate this by running canonicalisation only at publish time, not at every page render, and caching the resulting hash on the provenance record. The audit endpoint can become large on sites with many pages. Pagination plus the existing TYPO3 page cache handle the common case. For sites with hundreds of thousands of pages, a future version may introduce a dedicated cache backend, but this is not in scope for v0.1. The JSON-LD block adds bytes to every page, typically a few hundred. This is acceptable, and the block compresses well with gzip and brotli.

## 12. Security considerations

Private signing keys must never appear in the database, in the audit endpoint, in any JSON-LD block, or in any backup that is not encrypted at rest. The configuration documents this. The CLI commands check file permissions on the key path and refuse to operate if the private key is world-readable.

External identity URIs in `be_users` are user-supplied and must not be trusted as verified. The `verified` flag is set only by administrators with a designated permission. The JSON-LD output makes the verification status explicit so that consumers can decide how to weight it.

The audit endpoint is rate-limit-friendly. We recommend operators put the endpoint behind their existing rate limiting infrastructure. The endpoint itself does not implement rate limiting because TYPO3 has no canonical rate limiting layer to extend.

## 13. Testing strategy

The core ships with a functional test suite covering record creation, retrieval, rendering, and the events. The signing extension ships with cryptographic tests using known-answer test vectors for Ed25519. The audit extension ships with conformance tests that fetch the endpoint and validate the response against the specification schema. A separate repository hosts the cross-CMS conformance test suite, which any implementation can run against its own audit endpoint to claim specification conformance.

## 14. Documentation

Documentation lives in three places. The TYPO3 extension repository contains the standard TYPO3 documentation for installation, configuration, and developer API. The specification repository contains the protocol-level documentation. A short editorial guide lives separately and is aimed at content editors, explaining what a substantive revision is, when to add sources, and how to mark up facts. The editorial guide is the most underestimated piece of documentation in this project, because the value of the extensions is created by editors, not by developers.

## 15. Open questions for v0.2

Several decisions are deliberately deferred. Multiple authors per document via an MM table. Inheritance of provenance records, for example when a content element references another. Federation across sites so one audit endpoint can reference records hosted elsewhere. Verifiable Credentials integration for site keys via DIDs. C2PA manifest emission for documents that also include images. Editor user experience for fact markup beyond the initial dialog. Whether the audit endpoint should support a query interface.

We name these openly rather than pretending the v0.1 design covers them.

## 16. Project structure

The four extensions live in four repositories under a neutral GitHub organisation, ideally with TYPO3 Association involvement. The cross-CMS specification lives in a fifth repository. A sixth repository holds the conformance test suite. Each repository carries its own license (GPL-2.0-or-later for the extensions, as required for TYPO3 extensions in TER, CC-BY 4.0 for the specification), its own contribution guide, and its own release cadence. Releases of the core extension follow TYPO3 LTS cycles. Sub-extensions release independently but pin compatibility with the core.

A monorepo would simplify cross-extension changes during early development. We recommend resisting that simplification because the four-repo split is the signal that these are independent components, not a single product. The early friction is worth the long-term clarity.

## 17. Naming and branding

The TYPO3 extensions are named `provenance`, `provenance_signing`, `provenance_facts`, `provenance_audit`. No dkd prefix, no MX prefix, no marketing name. The specification is referenced as Content Provenance Specification or by its working version number. If the community later prefers a different name, the names move; the extensions are renamed without backward-compatibility shims, because v0.1 is explicitly not yet stable.

## 18. Timeline and milestones

The PRD does not prescribe a calendar. The phased plan from the parallel project plan applies. Concrete milestones at the artifact level.

**M1, core repository skeleton.** Repository created under neutral organisation, license, contribution guide, initial PRD and specification cross-link, empty extension skeleton with composer.json and Services.yaml in place.

**M2, core data model.** Tables, TCA, basic backend forms, no frontend rendering yet. Sufficient for a developer to create a record manually and inspect it.

**M3, core frontend rendering.** JSON-LD output on every page that has a record. PageRenderer hook in place. Backfill command working. Site reaches L2.

**M4, signing extension.** Key generation, key rotation, canonicalisation, signature generation, JSON-LD enrichment. Site can reach L3.

**M5, audit extension.** Well-known endpoint, pagination, public key exposure. Site can reach L4 minus facts.

**M6, facts extension.** Fact data model, RTE button, Microdata output, Web Annotation endpoint. Site reaches full L4.

**M7, documentation and editorial guide.** Sufficient for a non-dkd agency to install all four and produce a conformant site.

**M8, conformance test suite.** The first non-TYPO3 implementation (Drupal or WordPress) can run the tests against its own endpoint.

The expected order is M1 to M6 in sequence, with M7 and M8 running in parallel from M4 onward.

## 19. How to contribute

This is the section that most existing PRDs lack and that matters most for an open project. Contributors are welcome at three levels. Specification contributions go to the specification repository and follow the working group process. Code contributions to the TYPO3 extensions follow standard TYPO3 contribution conventions, with the core requiring two reviewer approvals and sub-extensions requiring one. Documentation and translation contributions are welcomed in any of the repositories and have a lower review bar.

A maintainers list is published in each repository and updated quarterly. Maintainership is added by existing maintainers based on sustained contribution. There is no formal corporate sponsorship of any maintainer position. dkd contributes maintainers like any other agency.

---

## Annex A, TCA fragments

The TCA fragments below are illustrative for v0.1. Production code will refine field types, eval rules, and access controls.

### tx_provenance_document

```php
return [
    'ctrl' => [
        'title' => 'LLL:EXT:provenance/Resources/Private/Language/locallang_db.xlf:tx_provenance_document',
        'label' => 'document_uri',
        'tstamp' => 'tstamp',
        'crdate' => 'crdate',
        'delete' => 'deleted',
        'enablecolumns' => ['disabled' => 'hidden'],
        'iconfile' => 'EXT:provenance/Resources/Public/Icons/document.svg',
    ],
    'columns' => [
        'document_uri' => [
            'label' => 'Document URI',
            'config' => [
                'type' => 'input',
                'size' => 50,
                'eval' => 'trim,unique',
                'required' => true,
                'readOnly' => true,
            ],
        ],
        'first_published_at' => [
            'label' => 'First published',
            'config' => [
                'type' => 'datetime',
                'format' => 'datetime',
                'readOnly' => true,
            ],
        ],
        'last_substantive_revision_at' => [
            'label' => 'Last substantive revision',
            'config' => [
                'type' => 'datetime',
                'format' => 'datetime',
            ],
        ],
        'authors' => [
            'label' => 'Authors',
            'config' => [
                'type' => 'group',
                'allowed' => 'be_users',
                'size' => 5,
                'minitems' => 1,
            ],
        ],
        'reviewer' => [
            'label' => 'Reviewer',
            'config' => [
                'type' => 'group',
                'allowed' => 'be_users',
                'size' => 1,
                'minitems' => 0,
                'maxitems' => 1,
            ],
        ],
        'sources' => [
            'label' => 'Sources',
            'config' => [
                'type' => 'inline',
                'foreign_table' => 'tx_provenance_source',
                'foreign_field' => 'document_uid',
                'appearance' => [
                    'collapseAll' => false,
                    'levelLinksPosition' => 'bottom',
                    'useSortable' => true,
                ],
            ],
        ],
        'content_hash' => [
            'label' => 'Content hash',
            'config' => [
                'type' => 'input',
                'size' => 64,
                'readOnly' => true,
            ],
        ],
        'signature' => [
            'label' => 'Signature',
            'config' => [
                'type' => 'text',
                'rows' => 3,
                'readOnly' => true,
            ],
        ],
        'signature_key_id' => [
            'label' => 'Key ID',
            'config' => [
                'type' => 'input',
                'size' => 30,
                'readOnly' => true,
            ],
        ],
        'conformance_level' => [
            'label' => 'Conformance level',
            'config' => [
                'type' => 'select',
                'renderType' => 'selectSingle',
                'items' => [
                    ['label' => 'L1 Discoverable', 'value' => 1],
                    ['label' => 'L2 Provenance', 'value' => 2],
                    ['label' => 'L3 Verifiable', 'value' => 3],
                    ['label' => 'L4 Citable', 'value' => 4],
                ],
                'default' => 2,
            ],
        ],
        'cms_version' => [
            'label' => 'CMS version',
            'config' => [
                'type' => 'input',
                'size' => 30,
                'readOnly' => true,
            ],
        ],
    ],
    'types' => [
        '0' => [
            'showitem' => 'document_uri, first_published_at, last_substantive_revision_at,
                          --div--;Authors, authors, reviewer,
                          --div--;Sources, sources,
                          --div--;Signature, content_hash, signature, signature_key_id,
                          --div--;Conformance, conformance_level, cms_version',
        ],
    ],
];
```

### tx_provenance_source

```php
return [
    'ctrl' => [
        'title' => 'Provenance Source',
        'label' => 'description',
        'sortby' => 'sorting',
        'tstamp' => 'tstamp',
        'crdate' => 'crdate',
        'delete' => 'deleted',
        'iconfile' => 'EXT:provenance/Resources/Public/Icons/source.svg',
    ],
    'columns' => [
        'document_uid' => [
            'config' => [
                'type' => 'passthrough',
            ],
        ],
        'url' => [
            'label' => 'Source URL',
            'config' => [
                'type' => 'link',
                'size' => 60,
                'required' => true,
            ],
        ],
        'description' => [
            'label' => 'What does this source support',
            'config' => [
                'type' => 'text',
                'rows' => 3,
                'eval' => 'trim',
                'required' => true,
            ],
        ],
        'accessed_at' => [
            'label' => 'Access date',
            'config' => [
                'type' => 'datetime',
                'format' => 'date',
                'required' => true,
            ],
        ],
        'archive_url' => [
            'label' => 'Archive URL (optional)',
            'config' => [
                'type' => 'link',
                'size' => 60,
            ],
        ],
    ],
    'types' => [
        '0' => [
            'showitem' => 'url, description, accessed_at, archive_url',
        ],
    ],
];
```

### tx_provenance_fact

```php
return [
    'ctrl' => [
        'title' => 'Provenance Fact',
        'label' => 'fact_id',
        'label_alt' => 'object_literal',
        'sortby' => 'sorting',
        'tstamp' => 'tstamp',
        'crdate' => 'crdate',
        'delete' => 'deleted',
        'enablecolumns' => ['disabled' => 'hidden'],
    ],
    'columns' => [
        'document_uid' => ['config' => ['type' => 'passthrough']],
        'tt_content_uid' => ['config' => ['type' => 'passthrough']],
        'fact_id' => [
            'label' => 'Fact ID',
            'config' => [
                'type' => 'input',
                'size' => 30,
                'eval' => 'trim,alphanum_x',
                'required' => true,
            ],
        ],
        'subject_uri' => [
            'label' => 'Subject',
            'config' => ['type' => 'input', 'size' => 60, 'eval' => 'trim', 'required' => true],
        ],
        'predicate' => [
            'label' => 'Predicate',
            'config' => ['type' => 'input', 'size' => 60, 'eval' => 'trim', 'required' => true],
        ],
        'object_literal' => [
            'label' => 'Object (literal value)',
            'config' => ['type' => 'input', 'size' => 60, 'eval' => 'trim'],
        ],
        'object_uri' => [
            'label' => 'Object (URI)',
            'config' => ['type' => 'input', 'size' => 60, 'eval' => 'trim'],
        ],
        'source_uid' => [
            'label' => 'Source',
            'config' => [
                'type' => 'select',
                'renderType' => 'selectSingle',
                'foreign_table' => 'tx_provenance_source',
                'foreign_table_where' => 'AND tx_provenance_source.document_uid = ###document_uid###',
                'minitems' => 0,
                'maxitems' => 1,
            ],
        ],
        'confidence' => [
            'label' => 'Confidence',
            'config' => [
                'type' => 'select',
                'renderType' => 'selectSingle',
                'items' => [
                    ['label' => 'Measured', 'value' => 'measured'],
                    ['label' => 'Estimated', 'value' => 'estimated'],
                    ['label' => 'Asserted', 'value' => 'asserted'],
                ],
                'default' => 'asserted',
            ],
        ],
        'valid_from' => [
            'config' => ['type' => 'datetime', 'format' => 'date'],
        ],
        'valid_until' => [
            'config' => ['type' => 'datetime', 'format' => 'date'],
        ],
    ],
    'types' => [
        '0' => [
            'showitem' => 'fact_id,
                          --div--;Statement, subject_uri, predicate, object_literal, object_uri,
                          --div--;Source, source_uid, confidence,
                          --div--;Validity, valid_from, valid_until',
        ],
    ],
];
```

### Additions to pages and tt_content

```php
$columns = [
    'provenance_document_uid' => [
        'label' => 'Provenance Record',
        'config' => [
            'type' => 'inline',
            'foreign_table' => 'tx_provenance_document',
            'minitems' => 0,
            'maxitems' => 1,
            'appearance' => [
                'collapseAll' => false,
                'expandSingle' => true,
            ],
        ],
    ],
];
```

Added to pages and tt_content via `ExtensionManagementUtility::addTCAcolumns()` plus `addToAllTCAtypes()` registering the column under a new "Provenance" tab.

### Additions to be_users

```php
$columns = [
    'external_identity_uri' => [
        'label' => 'External identity (ORCID, profile URL)',
        'config' => [
            'type' => 'input',
            'size' => 50,
            'eval' => 'trim',
        ],
    ],
    'verified' => [
        'label' => 'External identity verified',
        'config' => [
            'type' => 'check',
            'renderType' => 'checkboxToggle',
        ],
    ],
];
```

The `verified` column is editable only by users with the explicit permission `provenance.verify_identity`, enforced through a TCA field configuration permission and a DataHandler hook.

## Annex B, Out of scope for v0.1, named explicitly

Multiple authors via an MM table. Inheritance of provenance records when one content element references another. Federation, an audit endpoint referencing records hosted elsewhere. Verifiable Credentials integration for site keys via DIDs. C2PA manifest emission for mixed text and image documents. Editor user experience for fact markup beyond an initial dialog. A query interface on the audit endpoint. EXT:solr integration that consumes facts as indexed fields. None of these are wrong, all of them are deferred.
