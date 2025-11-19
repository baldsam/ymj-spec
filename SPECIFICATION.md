# YMJ File Format Specification (YAML · Markdown · JSON)

Version: 0.7 (Draft)
Status: Proposal
Maintainer: SFA

---

## 1. Purpose & Origin

YMJ is a machine-first, deterministic document format that combines:
- YAML header for structured identity and document semantics (about the doc)
- Markdown body for human-readable content (the payload)
- JSON footer for canonical operational metadata (how to handle the doc)

Origin: YMJ succeeds an internal prototype; we standardize on YMJ as the sole format.

Goals:
- Deterministic parsing with unambiguous structure
- Schema-driven ingestion and validation
- Strong, normalized metadata for indexing/graphing and operational behavior
- Durable for identity profiles, specs, knowledge pages, datasets, tickets, pages, traces, etc.

Non-goals:
- YMJ is not a database
- YMJ is not a container for binary assets

---

## 2. File Extension, Encoding, MIME

- Extension: `.ymj` (only)
- Encoding: UTF-8 (no BOM recommended)
- Line endings: `\n` preferred; `\r\n` tolerated
- Suggested MIME: `text/x-ymj` (custom) or `text/markdown` with tooling hints

---

## 3. Canonical Structure

A YMJ file MUST contain exactly three sections in order:

1) YAML Header (required)
- Surrounded by `---` fences (start and end)
- Declares document identity and semantics (about the doc)

2) Markdown Body (required)
- Free-form content; no structural requirements
- Tools MUST NOT infer behavior from body formatting

3) JSON Footer (required)
- Fenced with a Markdown code block labeled `json` (```json ... ```)
- Canonical operational metadata used for indexing and behavior

Authority Split
- YAML header: authoritative for identity/semantics
- JSON footer: authoritative for handling/behavior
- Markdown body: human narrative/content (non-normative)

Card Catalog Rules
- Line 1: exactly `---`
- Line 2: `doc_summary: <short summary>` (<=120 chars, single line, UTF-8)
- Fast index: tools MAY read only L1+L2 for a quick catalog; full validation still required

Markdown — Freeform (non-normative)
- No required banners or cross-section alignment
- Body may contain headings, prose, fenced blocks (csv/mermaid/code), images, etc.

---

## 4. Formal Requirements

YAML Header (required)
- Start fence: a line with exactly `---`
- YAML mapping content (UTF-8)
- Required keys (order recommended; parsers MUST NOT rely on order beyond `doc_summary` being the first key)
  - `doc_summary` (string, <=120 chars, single line) — SHOULD appear as the first YAML key (line 2)
  - `kind` (string): logical type (e.g., `identity_profile`, `note`, `spec`, `runbook`, `ticket`, `page`, `dataset`, `trace`, `policy`, `recipe`, `rfc`, `card`)
  - `version` (string): semver or document schema version for semantics
  - `subject` (string): canonical human title/name
  - `maintained_by` (string): owner (person/team)
- Recommended keys:
  - `ymj_spec` (string): YMJ spec version the document targets (e.g., `0.5`)
  - `validation_mode` (string): `strict|permissive` — document's intended enforcement profile\n  - `created` (ISO 8601), `updated` (ISO 8601)
  - `language` (BCP-47; e.g., `en-US`)
  - `classification` (public|internal|confidential|secret)
  - `attachments` (list): minimal catalog [{ uri, note?, sha256? }]
  - `provenance` (map): { source?, tool?, run_id? }
  - `payload_sha256` (string): SHA-256 of Markdown body (DEPRECATED in Header, moved to Footer)
- End fence: a line with exactly `---`

JSON Footer (required)
- Must be the last block in the file
- Fenced code block with `json` info string:
  - Strict mode: a line with exactly ```json begins, a line with exactly ``` ends
  - Permissive mode: accept ```json or ~~~json fences; still last block, still valid JSON
- JSON object fields (required):
  - `schema`: string/integer for JSON footer schema version (e.g. "1")
  - `payload_hash`: (string) SHA-256 of the Markdown body. Critical for validating state.
  - `index`: object with normalized fields for search/filter (keys implementation-defined)
    - Recommended: `tags` (array of strings), `title` (string)
    - Optional: `embedding` (array of floats) for semantic search.

- JSON object fields (optional):\r\n  - `schema_uri` (string URI): reference to a JSON Schema describing the JSON footer\r\n  - `ai` (object, optional): { `vector_ref`?:string, `embedding_id`?:string, `embedding_model`?:string }\r\n\r\nJSON object fields (optional handling directives under `ops`):\r\n  - `ops.body_format` (enum): `markdown|csv|mermaid|plain|json|yaml`
  - `ops.payload_mime` (string MIME) — DEPRECATED (advisory only; avoid in new documents). Authoritative directive is `ops.body_format`. If both present: Strict → reject on mismatch; Permissive → warn and ignore MIME; `ops.body_format` wins
  - `ops.body_selector` (string): how to locate payload in body (see §13)
  - `ops.attachments` (array): handling directives for attachments [{ uri?, select?, transform?, cache?, auth_ref? }]
  - `ops.handlers` (array): pipeline steps [{ name, args{} }]
  - `ops.routes` / `ops.page` (map): { route?, layout?, seo? } for web publishing
  - `ops.visibility` (public|private|role:[...])
  - `ops.retention` (map): { days?:int, expiry?:ISO 8601 }
  - `ops.priority` (low|normal|high)
  - `ops.cache` (map): { key?, etag? }
  - `ops.localization` (map): { locales:[string] }
  - `ops.chunking` (map): { chunk_id:int, total:int, prev_sha256?:string }
  - `ops.ticket` (map): { severity:int, code?:string, related_ids?:[string] }
  - `ops.trace` (map): { parent_id?:string, lineage_id?:string }
  - `ops.graph` (map): { edges:[{ type:string, to:string }] }\r\n  - `ops.body_encoding` (string): e.g., `base64`, `gzip`\r\n  - `ops.data_signature` (map): { alg:string, signature:string, key_id?:string }\r\n  - `ops.prompt_role` (string): `system|user|assistant|tool` (when used for prompting)\r\n  - `ops.auto_summary` (boolean): allow tooling to trust/generate machine summaries\r\n  - `ops.trace.revision` (string|int): explicit revision identifier
- Identity mirrors in JSON (forbidden):
  - Tools MUST NOT write/read JSON mirrors of YAML identity (e.g., `index.title`, `index.summary`, `index.owner`); reject or warn if present per validator mode

Attachments Reconciliation
- Effective attachments = YAML `attachments` ∪ JSON `ops.attachments`
- Precedence:
  - YAML is authoritative for `sha256` (identity)
  - JSON augments/overrides operational handling by `uri` match
  - JSON-only `uri` allowed (operational-only). In Strict mode, tools MAY warn if not listed in YAML catalog

---

## 5. Parsing & Detection

Primary Routing
- Match by extension `.ymj`

Content Heuristics (optional)
- If extension unknown, YMJ MAY be detected by:
  - Leading YAML front-matter fenced by `---` with valid YAML
  - Trailing fenced `json` (or `~~~json` in Permissive mode) code block with valid JSON

Error Handling
- Missing header or footer → reject
- Invalid JSON footer → reject
- Unparseable YAML header → reject

Security & Integrity
- Treat YMJ as untrusted input; never execute embedded content
- Avoid YAML features with implicit execution; no custom tags
- If `payload_sha256` exists, tools SHOULD verify it against the Markdown body using canonicalization: normalize line endings to `\n`, encode as UTF-8 (no BOM), and hash the exact body bytes (no trimming).
- If `attachments[].sha256` exists, tools SHOULD verify referenced content as raw bytes (no transcoding).
- If `ops.encryption` or `ops.signature` appear (future), do not store secrets; use key references only

---

## 6. Linking & Cross-References

WikiLinks
- Pattern: `[[PageTitle]]` within Markdown links to another page whose file stem matches `PageTitle`
- Tools SHOULD preserve links as written; link extraction scans Markdown body

Backlinks
- Compute backlinks from the parsed link graph across the corpus

---

## 7. Versioning

- YMJ uses semantic versioning for the spec
- `ymj_spec` (YAML, recommended) declares the YMJ spec version targeted by this document
- `version` (YAML) refers to the document schema for semantics
- `schema` (JSON) refers to the JSON-footer schema version

Non-normative note: A document may carry three versions with distinct roles: `ymj_spec` (spec the doc targets), `version` (doc's own semantic version), and `schema` (JSON footer schema). Example: `version: 2.1.0`, `ymj_spec: 0.6`, `schema: 1`.
\n---

## 8. Examples

Minimal YMJ
---
---
doc_summary: Minimal YMJ example
kind: note
version: 1.0.0
subject: Title
maintained_by: SFA
ymj_spec: 0.5
updated: 2025-10-13
---
# Title

Body text here.

```json
{"schema":"1","index":{"tags":["note"],"title":"Title"}}
```
---

CSV Payload with Selector
---
---
doc_summary: Sample data table (first rows inline; full file attached)
kind: dataset
version: 1.1.0
subject: sample-table
maintained_by: Data Team
attachments:
  - uri: file:///data/sample-table-full.csv
    sha256: "<sha256-of-full-csv>"
ymj_spec: 0.5
updated: 2025-10-13
---
# Sample Table

The first few rows:

```csv
col1,col2
alpha,1
beta,2
```

```json
{"schema":"1","index":{"tags":["dataset","csv"],"title":"sample-table"},"ops":{"body_format":"csv","body_selector":"first_fenced:csv"}}
```
---

Ticket / Error Log
---
---
doc_summary: Tool X failed to connect to service Y (timeout)
kind: ticket
version: 0.1.0
subject: TOOLX-ERR-1042
maintained_by: Core Ops
classification: internal
ymj_spec: 0.5
updated: 2025-10-13
---
# Incident Summary

Context, reproduction, and logs go here. Include fenced logs if useful.

```json
{"schema":"1","index":{"tags":["incident","ops"],"title":"TOOLX-ERR-1042"},"ops":{"ticket":{"severity":3,"code":"CONNECTION_TIMEOUT"},"trace":{"parent_id":"run-7c3f","lineage_id":"incident-20251013"}}}
```
---

---

## 9. Implementation Notes (sfa-brain)

Readers SHOULD:
- Accept `.ymj`
- For scanning, match extension OR detect by structure (header + json footer)
- Extract `[[WikiLinks]]` from Markdown body
- Respect JSON `ops.body_format`/`ops.body_selector` when extracting payloads

Writers SHOULD:
- Emit UTF-8 without BOM; prefer `\n` line endings
- Include YAML identity and JSON operational metadata; avoid identity mirroring in JSON
- Use attachments for large assets; keep body samples small when needed

---

## 10. Validation Rules (Machine-Only)

- Exactly one YAML header and one JSON footer MUST exist
- YAML header MUST parse and include required keys; parsers MUST NOT rely on key order (beyond `doc_summary` being the first key)
- `doc_summary` (YAML L2) MUST be <=120 chars, single line
- JSON footer MUST parse and include: `schema`, and `index`
- JSON MUST NOT include identity mirrors (title/summary/owner). Strict: reject; Permissive: warn and ignore
- If `ops.body_selector` is present, tools SHOULD validate the referenced block exists
- If `payload_sha256`/`attachments[].sha256` present, tools SHOULD verify hashes

---

## 11. Changelog

- 0.7 (2025-10-13): Add `validation_mode` (YAML), `schema_uri` and `ai` (JSON); extend ops.* with body_encoding/data_signature/prompt_role/auto_summary/trace.revision; add canonicalization rules; selector registry note\r\n- 0.6 (2025-10-13): Deprecate ops.payload_mime in favor of ops.body_format; add non-normative versioning note; warn on regex selectors; add optional validate_summary handler guidance
- 0.5 (2025-10-13): Relax YAML key order (doc_summary stays first); Strict vs Permissive footer fences; selector mini-language preview; attachments reconciliation; added ymj_spec
- 0.4 (2025-10-13): Markdown freeform; identity/handling split; attachments split (YAML catalog vs JSON ops); forbid JSON mirrors; add ops.*
- 0.3 (2025-10-13): Card Catalog rules; `doc_summary` required; initial status/tags in JSON index
- 0.2 (2025-10-13): Machine-first positioning; removed legacy/back-compat notes
- 0.1 (2025-10-13): Initial proposal

---

## 12. Applications & Profiles (Non-Normative)

These profiles illustrate typical uses of YMJ and the key fields that matter for each. They are guidance, not requirements. Implementations MUST follow the normative sections above; profiles exist to keep applications in mind during design.

- Context Profile (Agent/User/System)
  - kind: identity_profile
  - YAML focus: doc_summary, subject, maintained_by, updated
  - JSON ops: none required; index.tags: ["context"]
  - Linking: use Markdown [[WikiLinks]] for relationships

- Sub-Agent Context
  - kind: identity_profile
  - YAML focus: subject for agent id; provenance/tool/run_id recommended
  - JSON ops: index.tags: ["agent","context"]
  - Linking: to parent agent or controller via [[WikiLinks]]

- Dataset / CSV Payload
  - kind: dataset
  - YAML focus: attachments (uri, sha256), payload_sha256 optional
  - JSON ops: body_format: csv; body_selector: first_fenced:csv; index.tags: ["dataset","csv"]
  - Linking: to upstream recipe or source via [[WikiLinks]]

- Visualization (Mermaid/Graph)
  - kind: viz
  - YAML focus: subject, maintained_by
  - JSON ops: body_format: mermaid; handlers: [{name:"render_graph"}]
  - Linking: to data source or doc explaining the viz

- Ticket / Error Log
  - kind: ticket
  - YAML focus: classification, maintained_by, provenance (run_id)
  - JSON ops: ticket.{severity,code,related_ids}, trace.{parent_id,lineage_id}; index.tags: ["incident","ops"]
  - Linking: to affected components and follow-ups

- Web Page / Route
  - kind: page
  - YAML focus: subject (page title)
  - JSON ops: page.{route,layout,seo}; handlers optional
  - Linking: to related pages/sections via [[WikiLinks]]

- Runbook / Checklist
  - kind: runbook
  - YAML focus: subject, maintained_by
  - JSON ops: index.tags: ["ops","runbook"]
  - Linking: to tickets/incidents and systems

- API Doc / Mock
  - kind: spec
  - YAML focus: version (semver), subject
  - JSON ops: handlers: [{name:"render_api_doc"}] or body_format: json/yaml
  - Linking: to endpoints and client SDKs

- Experiment / Notebook
  - kind: experiment
  - YAML focus: provenance, updated, subject
  - JSON ops: index.tags: ["exp","metrics"], handlers for evaluation
  - Linking: to datasets/models

- Execution Trace
  - kind: trace
  - YAML focus: provenance (tool, run_id)
  - JSON ops: trace.{parent_id,lineage_id}, handlers for graphing
  - Linking: to inputs/outputs/tools

- Policy / Consent
  - kind: policy
  - YAML focus: classification, maintained_by, updated
  - JSON ops: index.tags for scope; visibility/retention optional
  - Linking: to systems/teams

- Knowledge Card
  - kind: card
  - YAML focus: doc_summary, subject (short), language optional
  - JSON ops: index.tags by topic
  - Linking: heavy use of [[WikiLinks]]; backlinks build context

- Dataset Manifest
  - kind: dataset
  - YAML focus: attachments (uri, sha256) catalog; payload_sha256 optional
  - JSON ops: handlers: [{name:"ingest_dataset"}], cache/retention optional
  - Linking: to schemas, consumers, lineage

- ETL Recipe
  - kind: recipe
  - YAML focus: subject, maintained_by, provenance
  - JSON ops: handlers: parse/transform/export steps; ops.attachments for sources
  - Linking: to datasets and outputs

Notes
- YAML: identity only; keep it declarative (about the doc)
- JSON: handling only; operational directives live under ops.*
- Markdown: freeform; place samples/snippets/tables as needed; extraction is guided by ops.body_format/body_selector

---

## 13. Body Selector Mini-Language (Preview)

Selectors allow deterministic extraction of payloads from freeform Markdown.

Syntax
- `first_fenced:<lang>` — first fenced code block with info string `<lang>`
- `nth_fenced:<lang>:<n>` — nth fenced block of type `<lang>` (1-based)
- `block_id:<id>` — a fenced block explicitly labeled or annotated by id (tooling-defined)
- `regex:<pattern>[/flags]` — first match group used to extract content (careful; validate)
- (Optional, future) `heading_path:<h1>/<h2>/... -> fenced:<lang>` — locate fence under a specific heading path

Precedence & Defaults
- Explicit `ops.body_selector` overrides defaults
- Default: if `ops.body_format` is set and selector omitted, use `first_fenced:<ops.body_format>`
- Tools SHOULD validate selector targets on read; STRICT mode may reject if missing
\r\nRegistry Note: Selector types are versioned. A lightweight registry (by type) SHOULD define patterns and validation for each selector. Tools MAY fetch registry info by `schema_uri` or embed defaults.\r\nSecurity note: Regex selectors are powerful but brittle and may introduce performance risks (ReDoS). Prefer structured selectors (`first_fenced`, `nth_fenced`, `block_id`). Strict mode SHOULD warn when `regex:` is used.\n
---

## 14. Tooling Responsibilities & Normalization

This section clarifies enforcement expectations and consistency behaviors for tools working with YMJ.

Enforcement Modes
- Strict mode:
  - Footer fences MUST be ```json only
  - JSON identity mirrors (index.title/summary/owner) MUST NOT be present → reject
  - Missing selector targets (when ops.body_selector is set) SHOULD cause rejection
  - JSON-only attachments not listed in YAML MAY warn; tools SHOULD allow operational-only URIs
- Permissive mode:
  - Footer fences MAY be ```json or ~~~json
  - JSON identity mirrors SHOULD warn (and be ignored)
  - Missing selector targets SHOULD warn
  - JSON-only attachments allowed with no warning

Normalization on Write
- Tools SHOULD write documents in Strict form when saving or emitting YMJ (canonical fences, no identity mirrors, doc_summary at line 2)
- Optional quality handler: `validate_summary` may derive a summary from the body and compare to YAML `doc_summary`; tools MAY warn on significant divergence.\n- Tools SHOULD preserve YAML attachments (identity catalog) and place handling-only attachments under JSON ops.attachments
- Tools SHOULD NOT infer behavior from Markdown; any extraction should follow ops.body_format/body_selector

Attachments Reconciliation (Implementation Notes)
- Effective set = YAML.attachments ∪ JSON.ops.attachments (by uri)
- YAML.sha256 is canonical for integrity checks when present
- JSON ops may add select/transform/cache/auth for the same uri; conflicts SHOULD favor JSON operational directives while preserving YAML identity

Binary Handling (Non-goal reiterated)
- Do not embed binary payloads in Markdown; prefer YAML attachments with sha256 and JSON ops for handling

Spec Versioning
- Tools SHOULD honor `ymj_spec` (if present) to select validation profiles; unknown versions fallback to latest permissive with warnings



