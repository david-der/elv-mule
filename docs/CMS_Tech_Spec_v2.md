# Agentic Printing — CMS Technical Specification

**Technical Design Document (v2)**
Elevance Health · Agentic Printing Initiative
April 21, 2026 · MVP Scope · Conceptual Data Model

*Companion to `CMS_PRD_v2.md`. This document describes the conceptual architecture of the CMS. It does not include SQL DDL or API endpoint schemas — those belong to the implementing engineering team. The goal here is to make the entity model, the pipelines, and the evaluation loop precise enough that implementation choices have an obvious right answer.*

---

## 1. Architectural Premise

The CMS is built on three premises the PRD makes explicit. This specification is how those premises become a system.

**Three layers of construction.** Structure is separate from Copy is separate from Data. Each has its own storage, its own lifecycle, its own governance. The render pipeline is the only place they combine, and they combine in a fixed phase order: structure first, copy second, slugs third.

**Versions AND variants.** A Managed Copy Block is not a versioned string. It is a two-dimensional grid: versions over time, variants across context. The storage model, the authoring UI, and the render-time resolution all operate on this grid. Treating variant as "just another string field" collapses the design into an unsound state. Treating it as a first-class entity is the spine of this specification.

**Measured reconstruction.** Every render is graded against the source letter it was deconstructed from. The grading is deterministic, two-track, and its result is persisted. This is what turns "we built templates" into "we built templates that round-trip with ≥98% fidelity."

**Ownership and change data capture.** Every asset in the CMS has an owner. Every action that changes state emits exactly one entry to an append-only, hash-chained audit log. Access is governed by a fixed role model with scope filters on ownership and variant dimension. These three concepts — ownership, role-based access, and the activity log — are the spine of the enterprise posture. They are designed in at the data-model and pipeline layers, not bolted on.

## 2. Conceptual Data Model

This section describes the domain entities and their relationships. Column lists, constraints, index strategy, and migration plans are explicitly excluded — they are implementation concerns and will be defined by the engineering team. The goal is to describe the model well enough that the implementation is mechanical.

### 2.1 Entities

**Structural Template.** The skeleton of a letter. It has a stable identity, a monotonic version number, a human-readable name, a correspondence type, and an ordered list of **Slots**. Templates live in a state machine: draft → published → retired. New versions are published, never mutated. A published template is the addressable unit the Render API consumes.

**Slot.** A position within a Structural Template. A slot has an ordinal (its position), a slot-type (greeting, body, disclosure, CTA, signature), an optional domain tag (medicare, medicaid, prior_auth, general), a required/optional flag, and a binding to a **Managed Copy Block** by canonical ID. A slot binds to the canonical, not to a specific variant — variant selection happens at render. A slot may optionally pin to a specific version (stability) or track the latest approved version (currency).

**Managed Copy Block.** A canonical unit of approved business content. Has a stable canonical identity, a human-readable name, a block-type, a domain tag, and a regulatory classification (none, hipaa, medicare, medicaid, state_mandate). A block itself holds no copy body — it is the container that owns the grid of cells.

**Block Cell.** This is the entity that holds actual copy. A cell is identified by the triple `(canonical_id, version, variant_id)`. The cell has a body (rich content with substitution slugs), a required-variables list derived from its body, an approval status, timestamps, and an author attribution. Cells are the unit of editing. Cells are never mutated — an edit produces a new cell with an incremented version.

**Variant.** A structured tag describing a slice of context. A variant has named **Dimensions** (line_of_business, jurisdiction, channel, audience, plan_type, language, correspondence_sensitivity) and, for each dimension it is scoped to, a value. A variant `{lob: medicaid, jurisdiction: VA}` leaves all other dimensions unconstrained — it matches any request whose `lob` is medicaid and whose `jurisdiction` is VA, regardless of channel or audience. Variants form a lattice: `{lob: medicaid}` is more general than `{lob: medicaid, jurisdiction: VA}`.

**Variant Dimension.** A named axis along which variants can differ. Dimensions are registered entities — the list is closed per version of the CMS. New dimensions require governance approval.

**Variable Context (transient).** A map of dot-paths to values supplied at render time. `member.first_name`, `claim.1.billed_amount`, `provider.npi`. Never persisted. Only its cryptographic hash is recorded on the Render Manifest.

**Render Manifest.** The immutable audit record of one render. Records template identity + version, every slot's resolved (canonical, version, variant), variable context hash, output URIs, output hashes, compliance scan ID and result, fidelity score, timestamps. Manifests are append-only.

**Fidelity Score.** The result of grading a rendered output against its source letter. Has two components — visual (Playwright) and semantic (DOCX/PDF) — and an overall pass/fail verdict. Bound to the Render Manifest.

**Rationalization Proposal.** An inbox item from the Deconstruction / Rationalization agents suggesting a new canonical block, a merge into an existing canonical, a reuse of an existing canonical, or a quarantine action. Governed through the approval queue.

**Approval Request.** A governance artifact. Points at either a proposed cell (new version of an existing variant, or a new variant), a proposed template version, or a proposed rationalization action. Holds approver identity (free-text in MVP, SSO-backed post-MVP), decision, and rationale. Immutable once decided.

**Source Letter.** A reference to an original .docx or .pdf letter that seeded a template through the deconstruction pipeline. Each Structural Template holds a link back to its source letter(s) — typically one-to-one but possibly many-to-one when rationalization merges similar letters into a single canonical template. Fidelity grading uses this link.

**User.** An identity in the CMS user directory. Has a stable identifier, a display name, a work email, an account status (active / suspended / retired), and a timestamp of last authentication. The MVP directory is standalone; post-MVP this entity federates to the Elevance IdP. A User holds zero or more Role Assignments.

**Role.** A named capability set from the fixed MVP role model (Viewer, Business Editor, Regulatory Content Owner, Content Governance Lead, Template Designer, Quality Lead, Render Consumer, Audit Reviewer, Access Administrator). Roles are registered entities; the set is closed for the MVP. Each role declares the set of Permissions it grants.

**Permission.** An atomic capability predicate. Examples: `block.cell.edit`, `block.cell.approve`, `template.publish`, `manifest.read`, `role.grant`, `audit_log.read`. Permissions are a closed enumerated set. Roles are defined as bags of Permissions.

**Role Assignment.** A binding of a User to a Role with an optional Scope Filter. The Scope Filter restricts the assignment by block ownership class (e.g., `owning_role = regulatory`) and/or by variant dimension values (e.g., `{lob: medicaid}` — only variants matching this slice are in scope for this assignment). Assignments carry the granting actor, the justification, and start/end timestamps. An expired or revoked assignment leaves a durable record; it is never deleted.

**Ownership.** A first-class relationship expressing which role is accountable for which asset. A Managed Copy Block has one `owning_role` for its copy as a whole, and zero or more per-variant `stewardship` assignments (e.g., Medicare sub-team stewards all `{lob: medicare}` cells). A Structural Template has one `owning_team`. A Render Manifest has a `requesting_system` ownership. Ownership transfers are a logged, approved action — not a free-form edit.

**Audit Event.** The primary CDC entity. Every state-changing action in the CMS produces exactly one Audit Event. An event carries: event identifier (UUID + monotonic sequence), actor (User identifier and effective role set at the time of action), wall-clock timestamp (UTC, millisecond precision) and logical timestamp (monotonic sequence), action type (from a closed enumeration), target entity type + canonical identifier + version/variant when applicable, from-state and to-state (for state transitions), payload diff (structured delta, PHI-scrubbed), reason code and free-text justification, correlation identifier (groups related events in a governance flow), event hash, previous event hash (chains the log for tamper evidence). Audit Events are append-only. No event is ever mutated after write. The log is the sole writer of this entity; no other pipeline or UI surface has write access.

**Access Request.** A governance artifact created when an editor attempts an action their current scope does not permit and chooses to escalate. Holds the requesting user, the attempted action, the missing permission/scope, the business justification, and a decision (approve / deny / route-to-governance-lead) with reviewer identity. Resolution emits an Audit Event.

### 2.2 Relationships

| From | Cardinality | To | Notes |
|---|---|---|---|
| Structural Template | 1 → n | Slot | Ordered slots within a template. |
| Slot | n → 1 | Managed Copy Block (by canonical_id) | Slot binds to canonical, not a specific cell. |
| Managed Copy Block | 1 → n | Block Cell | A block is a grid of cells. |
| Block Cell | n → 1 | Variant | Each cell has exactly one variant. |
| Variant | n ↔ n | Variant Dimension | Through Variant Dimension Values. |
| Structural Template | 1 → n | Source Letter | Links back to originating letter(s) for fidelity grading. |
| Render Manifest | n → 1 | Structural Template (specific version) | Manifest records the exact template version used. |
| Render Manifest | 1 → n | Slot Resolution | One per slot, records canonical + version + variant. |
| Render Manifest | 1 → 1 | Fidelity Score | When grading ran. |
| Approval Request | n → 1 | Block Cell \| Template \| Proposal | Polymorphic target. |
| Rationalization | n → 0..n | Managed Copy Block | Derived-from lineage; append-only. |
| User | 1 → n | Role Assignment | A user holds zero-or-more role assignments. |
| Role Assignment | n → 1 | Role | Assignment binds a user to a role with a scope filter. |
| Role | n ↔ n | Permission | Roles are bags of permissions (join table). |
| Managed Copy Block | n → 1 | Role (`owning_role`) | Every block has one accountable role. |
| Managed Copy Block | 1 → n | Ownership (stewardship) | Per-variant stewardship assignments. |
| Structural Template | n → 1 | Team (owning) | Every template has an accountable team. |
| Audit Event | n → 1 | User (actor) | Every event names an actor. |
| Audit Event | n → 1 | Target entity (polymorphic) | Event targets block/cell/variant/template/manifest/role/assignment. |
| Audit Event | n → 1 | Audit Event (previous) | Hash chain: every event references its predecessor's hash. |
| Access Request | n → 1 | User (requester) | A user raises the request. |
| Access Request | n → 1 | User (decider) | An Access Administrator or Governance Lead decides. |

The key structural claim: **the Block Cell table is the heart of the CMS.** Rows in the cell table are keyed on `(canonical_id, version, variant_id)`. Uniqueness is enforced on this triple. Reads are by canonical_id (list all cells for a block) or by (canonical_id, variant_id) ordered by version descending (give me the latest cell for this variant). Writes are insert-only — editing creates a new row.

### 2.3 The Variant Model in Detail

The variant grid is the structural decision that makes the rest of the system coherent. A worked example.

*Suppose `appeal_rights_disclosure` is a canonical block.* Its Variant Dimensions in scope are `line_of_business` and `jurisdiction`. The block has cells at the following variants:

| Variant | Latest Version | Status |
|---|---|---|
| `{national}` | v2 | approved |
| `{lob: medicare}` | v3 | approved |
| `{lob: medicaid}` | v5 | approved |
| `{lob: medicaid, jurisdiction: VA}` | v4 | approved |
| `{lob: medicaid, jurisdiction: CA}` | v6 | approved |
| `{lob: medicaid, jurisdiction: TX}` | v1 | proposed |
| `{lob: commercial}` | v2 | approved |

At render time the CMS receives a variable context that implies, among other things, `line_of_business = medicaid` and `member.state = VA`. The resolution ranks candidate variants by specificity:

1. `{lob: medicaid, jurisdiction: VA}` — matches on 2 dimensions. **Winner.**
2. `{lob: medicaid}` — matches on 1 dimension.
3. `{national}` — the universal fallback.

The cell at `{lob: medicaid, jurisdiction: VA}`, version 4, is selected. Its body is hydrated and its slugs are substituted. The Render Manifest records `appeal_rights_disclosure / v4 / {lob: medicaid, jurisdiction: VA}`.

Now suppose a render arrives with `line_of_business = medicaid, member.state = TX`. The TX cell exists but is `proposed`, not `approved`. It is excluded. The resolution falls back to `{lob: medicaid}` v5. The manifest records that cell. This behavior — "proposed cells are invisible to production renders" — is the whole point of the approval lifecycle.

### 2.4 Why Postgres

Relational integrity matters here in a way that NoSQL does not serve cleanly:

- The cell table's uniqueness constraint on `(canonical_id, version, variant_id)` is a natural relational constraint; emulating it in a key-value store is fragile.
- Variant resolution is a ranked query across a set of rows with a specificity score — efficiently expressible as SQL with a recursive CTE or a ranked join; awkward in document stores.
- Lineage (`derived_from`) is a graph traversal; while graph DBs are better, Postgres with recursive CTEs handles MVP scale easily.
- The Render Manifest's slot resolutions are a fixed-shape child table with FKs back to cells; this is exactly what relational is for.
- Governance audit requires referential integrity between approvals, cells, and users; Postgres enforces it.

A read replica + Redis cache on the block resolution hot path handles MVP throughput. Vector search (for semantic block lookup in the editor) is delegated to a pgvector extension rather than a separate OpenSearch instance — one less system.

### 2.5 What is intentionally not modeled here

No column types. No indexes. No partitioning strategy. No migration plan. No retention policy in days. These are implementation decisions. The data model described above is sufficient for engineering to produce DDL and to reason about query patterns; nothing here dictates specific storage choices beyond the relational-integrity argument.

## 3. Access Control and Change Data Capture

This section specifies how ownership, authorization, and the audit log are wired into the system. It sits between the data model (§2) and the pipelines (§4–§6) because every pipeline emits audit events and every pipeline is subject to access checks.

### 3.1 Access control model

The MVP ships a fixed role model with a small number of roles and a closed set of permissions. Authorization decisions are deterministic: given a user's role assignments and the target of an attempted action, the system produces a single `permit | deny` verdict with a reason.

**Evaluation order.**

1. Resolve the caller to a User. For UI calls, this is the authenticated session; for Render API calls, it is the calling system's service identity, which is itself a User of type `system`.
2. Collect the User's active Role Assignments (non-expired, non-revoked). Each assignment contributes its Role's Permissions plus its Scope Filter.
3. For the action being attempted, identify the required Permission. Every CMS action maps to exactly one required permission, documented in the permission catalog.
4. If no assignment contributes the required permission → `deny(missing_permission)`. If any assignment contributes it → proceed to scope check.
5. Evaluate the Scope Filter against the target. Scope filters are pattern predicates: `owning_role = regulatory` matches blocks owned by the regulatory role; `variant.lob = medicaid` matches cells at variants with `lob = medicaid`. An empty scope filter means "no restriction." Multiple filters on the same assignment are conjunctive; assignments are disjunctive across each other.
6. If at least one assignment's scope filter matches → `permit`. Otherwise → `deny(out_of_scope)`.

Both verdicts — permit and deny — emit an Audit Event. The deny event carries the attempted action, the missing permission or mismatched scope, and the user's effective role set at the time. This is what makes unauthorized attempts observable.

**Scope filter semantics.** Filters are specified as key-value pairs over two namespaces: `ownership.*` (properties of the target's ownership record) and `variant.*` (properties of the target's variant tag, when applicable). A filter key that does not apply to the target type (e.g., `variant.lob` applied to a Structural Template) is treated as vacuously true — the assignment grants on that target when other filters match. This keeps role definitions simple without producing surprise behavior.

**Derived permissions for aggregated targets.** Some actions target an aggregate (e.g., fan-out edit across 50 cells). The aggregate permit requires permit on *every* member cell. A partial permit is treated as deny; fan-out is all-or-nothing. This is intentional — partial fan-outs would produce states that are expensive to reason about in audit.

**Service identities.** Downstream systems that call the Render API authenticate as `system` users with Render Consumer role and a scope filter naming the templates they may render. This gives the activity log a clean actor for machine-initiated renders.

### 3.2 Ownership as a data-model constraint

Ownership is not metadata — it is a validity invariant enforced by the data model:

- A Managed Copy Block without a valid active `owning_role` is invalid. Insert/update transactions that would produce such a state are rejected.
- A Structural Template without a valid active `owning_team` is invalid, same enforcement.
- Transferring ownership is an atomic action routed through a specific endpoint. The old owner is written to an `ownership_history` append-only table, the new owner is set, and an Audit Event is emitted with the correlation ID of the transfer.
- When a User is suspended or retired, the system scans for assets where they are the sole steward; those assets enter a `needs_ownership` state and surface to the Access Administrator's inbox. Renders against blocks in this state continue to work (reads are unaffected); writes (new cell versions) are blocked until ownership is reassigned.

This behavior eliminates the silent-drift failure mode where a departing employee leaves a block nobody realizes is orphaned until a regulator asks about it.

### 3.3 The Audit Event store

The Audit Event entity is the single source of truth for the activity log. Its storage has three properties that together make it trustworthy.

**Append-only.** The Audit Event table accepts only inserts. The implementation enforces this at the database grant level — no role, including the application's primary writer role, holds UPDATE or DELETE privilege on this table. Attempted writes from any other path are a structural error.

**Hash-chained.** Each event's hash is computed over (`event_id, actor, timestamp_logical, action, target, from_state, to_state, payload_diff, reason, correlation_id, previous_event_hash`). The previous_event_hash is the hash of the immediately preceding event in monotonic sequence order. Breaking the chain at any point — by tampering with any field of any past event — invalidates every hash from that point forward. A periodic verifier walks the chain and reports integrity; it is also run inline on Audit Reviewer query responses at a small performance cost that is accepted for audit trust.

**Causally ordered.** Each event carries both a wall-clock timestamp (for human reading) and a monotonic sequence number (for ordering). Sequence is assigned at commit time by a sequence owned by the log table. Replaying a correlation-id's events in sequence order reconstructs the exact flow of an operation regardless of wall-clock skew across application instances.

**What is captured.** The action catalog is closed and enumerated. The MVP catalog covers: block create / retire, cell create / edit / promote / rollback / retire, variant dimension add / retire / populate, template create / slot bind / slot unbind / publish / deprecate, approval submit / approve / reject / comment, render execute / fail / held_for_compliance, fidelity score record, permission grant / revoke, role assignment create / modify / expire, ownership transfer, rationalization proposal intake / accept / reject, access request submit / decide, authentication success / failure. Read-only actions (viewing a block, browsing the manifest, running a query) do not emit events. This keeps the log's signal density high.

**Payload diff format.** For edit-type events, the payload diff is a structured JSON object naming the fields that changed, their before-values and after-values, with a consistent PHI-scrub pass applied. The scrub pass is schema-driven: specific field paths (e.g., anything under `context.member.*`) are replaced by their hashes. The scrub list is maintained alongside the payload schema registry and reviewed with every schema change.

**Retention and archival.** Events older than 12 months are moved to a cold partition; older than 7 years, to archival storage. Queries still work transparently across partitions; the 99th-percentile query latency of the Activity Log surface is budgeted to a ceiling that the partition plan respects.

**Export.** Any authorized Audit Reviewer can export a query result as a PDF (for human consumption) and a signed JSON bundle (for inter-system audit). The signed bundle includes the last event's hash and a validator-ready manifest so an external auditor can verify chain integrity without CMS access.

### 3.4 How pipelines emit events

Every pipeline in §4–§6 that changes state emits its own Audit Event. The emission is a hard part of the transaction boundary: the state change and the event insert commit together or roll back together. This is enforced in code and tested by a chaos-testing harness that introduces mid-transaction faults and verifies no state change leaves the system without a corresponding event.

The render pipeline (§5) is the one place this contract is most visible: a successful render writes a Render Manifest *and* a `render.succeeded` Audit Event within the same transaction; a held-for-compliance render writes the Manifest in `held` state *and* a `render.held_for_compliance` event. Consumers of the activity log can rely on the invariant that every manifest has a corresponding render event and vice versa.

### 3.5 Relationship to Render Manifests

The Render Manifest and the Audit Event are two different constructs with overlapping concerns. The manifest is the immutable record of what a render produced — what template, what cells, what output hash, what compliance result. The audit event is the immutable record of a *state transition* — who triggered the render, when, with what role, and in what governance context.

The two are linked: the render event carries the manifest ID, and the manifest carries the correlation ID that groups its render event with the downstream fidelity-score event and any subsequent hold-and-release events. This means a single drill-down from either side reconstructs the full picture. Audit Reviewers start from the manifest when a regulator asks "what did we send this member"; they start from the activity log when a security review asks "what did this user do last week."

### 3.6 What this section does not specify

Algorithms, table definitions, index strategies, and partition schemes are implementation concerns. The claims that must survive from this section into the implementation are: every state change emits exactly one event; the event store is append-only and hash-chained; every action is authorized against a role and a scope filter; and ownership is a validity invariant rather than a soft metadata field.

## 4. The Deconstruction Pipeline

Source letters enter the CMS as .docx or .pdf files. The Deconstruction Agent (Agent 02) is the canonical front door. This section describes the pipeline the CMS interacts with on the ingest side.

### 3.1 Phases

**Phase 1 — Layout extraction.** The source document is parsed into a tree of layout elements: pages, headers, footers, sections, paragraphs, tables, images. Each element carries geometry (bounding box, page number), style (font, size, weight, color), and content.

**Phase 2 — Layer classification.** Each element is classified as `structural`, `cms_content`, `variable`, or `mixed`. Structural elements become part of the Structural Template skeleton. cms_content elements become proposed Managed Copy Blocks. Variable elements become substitution slugs. Mixed elements are split.

**Phase 3 — Variable slug extraction.** Within cms_content, member-identifying text is identified (names, IDs, dates, amounts, addresses) and replaced with typed slugs in the appropriate namespace: `{{member.first_name}}`, `{{claim.1.billed_amount | currency}}`, etc. The element's "required_variables" list is computed.

**Phase 4 — Rationalization proposal.** Every proposed block is compared via semantic similarity (pgvector embeddings) against the existing canonical library. The Rationalization Agent (Agent 01) produces a proposal per block: REUSE (this is an existing canonical; reuse it), MERGE (this is near-identical to an existing canonical; promote it to a new version of that canonical), CREATE (truly new; allocate a new canonical_id), QUARANTINE (ambiguous; human must decide).

**Phase 5 — Governance intake.** Proposals land in the Approval Queue. A human governs the merge/create/quarantine decisions. REUSE is auto-accepted. Approved blocks are inserted into the canonical library as new cells; approved template assembly becomes a new Structural Template draft.

**Phase 6 — Template binding.** The Template Builder surface (governance UI) lets a Template Designer confirm the slot-to-block bindings proposed by Deconstruction, adjust them, and publish the Structural Template. The source letter ID is retained as a link on the template for fidelity grading.

### 3.2 What the CMS stores

The CMS is the downstream of this pipeline. It holds:

- The Structural Template produced by Phase 6.
- The proposed or approved Block Cells from Phases 4–5.
- The Rationalization Proposals in their governance state.
- A link from each Structural Template back to its Source Letter (an S3 object, or equivalent immutable blob store).

The CMS does not store raw layout trees. Those live in the Deconstruction Agent's workspace and are superseded once the template is published.

## 5. The Reconstruction / Render Pipeline

Render is the inverse of deconstruction. It combines the three layers in phase order.

### 4.1 Phases

**Phase 1 — Template load.** The Render API receives `{template_id, template_version?, variable_context, output_formats[]}`. The specified template (or the latest published version if unspecified) is loaded with its slot list.

**Phase 2 — Slot resolution.** For each slot, the bound canonical_id is resolved against the variable_context to select the winning Block Cell (`canonical_id, version, variant`). The algorithm is described in §2.3 and §5.2. Failure modes: no matching variant and no fallback → render error `NO_VARIANT_MATCH`; variant found but proposed/draft status → falls back or errors per strict mode.

**Phase 3 — Copy hydration.** The body of each resolved cell is inserted into its slot. The output at this phase is a fully-assembled document with all structural elements and all managed copy in place, but with substitution slugs still unresolved.

**Phase 4 — Slug substitution.** Every slug in the hydrated document is resolved against the variable_context. Format directives (`| currency`, `| date_long`, `| mask_ssn`, etc.) run at this phase. Failure modes: unresolved variable in strict mode → `UNRESOLVED_VARIABLE`; directive error → `DIRECTIVE_FORMAT_ERROR`.

**Phase 5 — Format materialization.** The phase-4 output is materialized into the requested output formats: .docx via the docx skill, .pdf via a tagged-PDF pipeline, accessible HTML via the HTML emitter. All three are semantically equivalent; the .pdf is the regulator-facing artifact, .docx is the editable artifact, HTML is the digital-delivery and accessibility surface.

**Phase 6 — Compliance gate.** The 508(c) Compliance Agent runs post-render. Any Critical or Major finding holds the release — outputs are produced but not returned. A governance-gated force-release path exists with an explicit override reason recorded.

**Phase 7 — Fidelity grading** (when applicable). If the render was triggered against a template that has a linked Source Letter, the grader runs. See §6.

**Phase 8 — Manifest emission.** The Render Manifest is written with every slot resolution, the variable_context hash, output hashes, compliance result, and fidelity score. Variable context values are discarded.

### 4.2 Variant Resolution Algorithm

Given a set of populated variants for a block and a request context, the algorithm:

1. Discard variants that require dimensions not present in the request context (e.g., a variant that specifies a jurisdiction when the request does not declare one).
2. Discard variants whose dimension values disagree with the request (e.g., `jurisdiction = CA` when the request says `jurisdiction = VA`).
3. Of the remaining, score each by the number of matching dimensions — higher is more specific.
4. Ties broken by a fixed dimension priority order (jurisdiction > line_of_business > channel > ...). The priority order is a governed constant, documented alongside the dimension registry.
5. The highest-scoring variant wins. Its latest approved Block Cell is the resolved cell.
6. If no variants remain, the `{national}` or declared default variant is used. If no default exists, the render fails with `NO_VARIANT_MATCH`.

This algorithm is deterministic. Given the same inputs it produces the same output. It is reproducible from a Render Manifest: the manifest records which variant won, so replaying with the same context produces the identical selection without re-running the algorithm.

### 4.3 Determinism

Rendering is deterministic when `(template_id, template_version, block_cell_set, variable_context)` are fixed. "Block cell set" here is the actual set of cells resolved, not the set available — because a block may have had a new cell added after the render. Reconstruction from a manifest uses the cells the manifest names, not the latest cells. This is why the manifest records `(canonical_id, version, variant)` per slot and not just the canonical.

## 6. The Grading / Evaluator Pipeline

Grading is the closed loop that makes reconstruction a measured claim rather than an asserted one.

### 5.1 What gets graded, when

- Every render produced in **calibration mode** is graded against the Structural Template's linked Source Letter.
- Every template publishes with a baseline grade (the grade of its calibration render using the synthetic context that matches the source letter's original data).
- Every edit to a Block Cell, when promoted to approved, triggers a regression grading pass on every Structural Template that uses the affected block. A grade that drops below threshold holds the promotion until the Quality Lead reviews.

Production renders (calls from downstream consumers with live variable contexts) are not graded against a source letter — there is no source to grade against for a personalized letter. They are still subject to the post-render 508(c) compliance gate.

### 5.2 Visual Track (Playwright)

The visual evaluator treats the rendered output as a set of images and compares them to images of the source letter.

Both documents are converted to high-DPI images — page-by-page for PDFs, at representative viewport widths for HTML. Playwright drives a headless Chromium, loads each, screenshots at a fixed DPI. The comparison is a structural image similarity (SSIM) plus a band-limited perceptual diff that ignores sub-pixel font rendering variance.

The evaluator produces:
- A per-page SSIM score.
- A perceptual diff image highlighting the regions where the two documents differ.
- An overall visual score 0–100 (a weighted aggregate).

**What visual is good at.** Catching layout breaks: a misaligned table, a missing logo, a section that landed on the wrong page, a heading that lost its styling. Catching font changes. Catching color drift.

**What visual is bad at.** Judging whether the copy is correct. Two letters that say completely different things can be visually indistinguishable if the layout is the same.

### 5.3 Semantic Track (DOCX / PDF Skills)

The semantic evaluator treats both documents as structured content.

The PDF skill (for both .pdf inputs) and the DOCX skill (for .docx inputs) extract:
- Full text, paragraph-by-paragraph.
- Heading structure (levels H1, H2, H3…).
- List structure (ordered / unordered, nesting).
- Table structure (header rows, merged cells, data cells).
- Hyperlink targets and their display text.
- Image alt text.
- Document metadata (title, language, author).

The evaluator compares these structures:
- **Text comparison** is normalized — substitution slugs in the reconstruction are re-substituted with the source letter's original values before comparison. After normalization, the text diff should be near-empty.
- **Heading structure** must match exactly. A missing or re-leveled heading is a critical finding.
- **Tables** must match on header count, header text, row count; per-cell text comparison at the same normalization.
- **Alt text and metadata** must match for accessibility parity.

The evaluator produces:
- A per-category score (text, heading, list, table, link, image, metadata).
- An aggregate semantic score 0–100 with category weights documented alongside the evaluator.
- A structured diff report enumerating every mismatch.

**What semantic is good at.** Catching copy substitutions, lost headings, broken tables, missing alt text, accessibility regressions, incorrect metadata.

**What semantic is bad at.** Judging visual fidelity. A letter with semantically perfect content but wrong page layout will pass semantically while failing visually.

### 5.4 Composite Verdict

The two tracks are complementary. A template reaches reconstruction threshold only when **both** tracks exceed their thresholds: visual ≥ 95 and semantic ≥ 98. These thresholds are tunable but conservative; they are calibrated against the 75-letter seed corpus and locked for the MVP.

A failed grading produces an inline report in the Approval Queue and the Fidelity Dashboard. The report includes both tracks' findings, a side-by-side rendering, and a "promote anyway with override reason" path for edge cases where the grader's conservatism conflicts with a correct output (e.g., an intentional layout improvement that the baseline does not match).

### 5.5 Why both tracks, why not one

A single-track evaluator is either too strict (visual-only pixel diff flags harmless font rendering drift as regressions) or too weak (text-only diff misses layout breaks entirely). Running both in parallel, and requiring both to pass, gives the grading loop a useful signal-to-noise ratio. Each track has predictable failure modes and predictable false positives. The Quality Lead learns those modes in the first weeks of MVP and tunes thresholds accordingly.

## 7. Governance, Approvals, and Lifecycle

### 6.1 The Block Cell Lifecycle

Block Cells move through the lifecycle: `draft → proposed → approved → retired`. Transitions are one-way; an approved cell cannot return to draft. A cell is retired by promoting a new cell in its variant — the old cell remains queryable for audit but is no longer selected by the render pipeline.

The 508(c) Compliance Agent pre-scans every `draft → proposed` transition. Critical findings block the transition. Approvals (`proposed → approved`) require a passing scan and a governance decision. Decisions carry a reviewer identity (a User record from the identity plane), a timestamp, and a rationale. Every transition emits an Audit Event with the transition's correlation ID so the full proposal-scan-decision-promotion chain can be replayed from the log.

### 6.2 The Structural Template Lifecycle

Templates move through `draft → published → retired`. A draft can be edited freely. Publishing requires every required slot to be bound. A published template is immutable — changes produce a new template version. Retirement prevents new renders from picking the template but preserves historical renders and their manifests.

### 6.3 Approval Queue

The queue is a single inbox. Items:
- Proposed Block Cells (new versions of existing variants, and new variants).
- Proposed Structural Template versions.
- Rationalization Proposals from the Deconstruction/Rationalization agents.
- Fidelity Regressions — any promotion that would drop a template below threshold.

Each item carries enough context for the reviewer to decide without opening another screen: diff view, dependents list, compliance scan result, fidelity delta. Bulk actions are supported for REUSE proposals (auto-approve recommended; reviewer can dismiss). Cross-variant fan-out edits are a single approval item, not 50.

## 8. Integrations

### 7.1 Deconstruction Agent (Agent 02)

**Direction:** into the CMS.
**Contract:** Deconstruction emits a Structural Template proposal (with slot-to-block bindings) and a set of Block Cell proposals, each with a proposed canonical binding action (REUSE / MERGE / CREATE / QUARANTINE) from the Rationalization Agent. Delivery via a domain event on the `argus.cms.*` event bus and a companion S3 blob holding the proposal payload.

### 7.2 Rationalization Agent (Agent 01)

**Direction:** mediated — runs inside the deconstruction pipeline but also runs periodically on the existing canonical library to propose merges.
**Contract:** emits Rationalization Proposals into the Approval Queue. Accepts feedback on its past proposals (the Governance Lead's accept/reject decisions) for active learning.

### 7.3 508(c) Compliance Agent (Agent 03 / 04)

**Direction:** mediated — called synchronously by the CMS.
**Contract:** two entry points. `scan(cell_body, context)` for pre-save guardrails — runs within the editor save path. `scan_render(rendered_output)` for post-render gates — runs within the render pipeline. Returns a structured finding set: Critical, Major, Minor with WCAG references, locations, recommended remediations.

### 7.4 Render Consumers

**Direction:** into the CMS.
**Contract:** downstream systems call the Render API with a template ID and a variable context. Synchronous response for single renders; batch API for bulk (batch is a single-mode MVP, scale-out post-MVP). Response includes output URIs, Render Manifest ID, and compliance status.

### 7.5 Source Systems

**Direction:** into the render pipeline, ephemerally.
**Contract:** the variable context is sourced from whatever system owns the relevant namespace — member from the enrollment/member system, claim from claims, provider from provider data. This is a contract at the caller's layer, not the CMS's — the CMS receives a composed variable context, not raw source-system records. The CMS hashes the context and discards it.

## 9. Non-Functional Requirements

Numeric MVP targets are deferred until calibration against the seed corpus establishes achievable baselines. Non-negotiables are called out explicitly.

- **PHI durability:** zero. No variable context values in any durable store. Only hashes. *(Non-negotiable.)*
- **Encryption:** at rest and in transit for all CMS data. *(Non-negotiable.)*
- **Render determinism:** given the same `(template version, block cell set, variable context)`, produce byte-identical outputs modulo embedded timestamps. *(Non-negotiable.)*
- **Reconstructibility:** every past manifest can re-render with the original context and produce byte-identical outputs. *(Non-negotiable on MVP corpus; post-MVP scale target is 100%.)*
- **Audit retention:** Render Manifests, Block Cell versions, and Audit Events retained 7 years online (or partitioned-cold with transparent query), archival beyond. *(Regulatory, non-negotiable.)*
- **Accessibility:** every rendered PDF passes an external accessibility validator (axe / Adobe). *(Non-negotiable.)*
- **Audit completeness:** every state-changing action produces exactly one Audit Event, committed within the same transaction as the state change. Verified by chaos-testing harness. *(Non-negotiable.)*
- **Audit immutability:** the Audit Event table holds only INSERT privileges for the application writer role; no UPDATE or DELETE privilege exists on that table for any role. Chain-verification runs nightly and on demand. *(Non-negotiable.)*
- **Access enforcement:** every write action is authorized against the role model and scope filter before it reaches the data layer. No action in the system bypasses authorization. *(Non-negotiable.)*
- **Ownership validity:** blocks and templates without a valid active owner enter a quarantined state that blocks writes. Reads remain available. *(Non-negotiable.)*
- **Render latency:** TBD after calibration. *(Per MVP scope discipline, specific numbers are set from data, not from guess.)*
- **Throughput:** TBD after calibration.
- **API availability:** MVP-environment SLA, not production.

## 10. MVP Implementation Sequence

The PRD's milestones map to technical sequences:

- **M1 — Data plane + identity.** Postgres schema is landed. Block Cells, Templates, Variants, Dimensions, Manifests, Users, Roles, Permissions, Role Assignments, Ownership, and the Audit Event table exist. CRUD endpoints for blocks and templates wrap every write in an authorization check and an Audit Event emission. Variant resolution algorithm is implemented and covered by unit tests. The Audit Event table is created with INSERT-only grants; chain verification is implemented and smoke-tested. Seed corpus of canonical blocks loaded from the existing `Letter_*_DECOMPOSED.html` artifacts. Seed user directory and role assignments created for MVP personas.

- **M2 — Render pipeline.** The sync render API is operational. The docx skill, a PDF composer, and the HTML emitter are wired. The grading evaluator's visual (Playwright) and semantic (DOCX/PDF skill) tracks are implemented and producing scores. Render events are emitted to the Audit Event store within the manifest-write transaction.

- **M3 — Compliance integration.** The 508(c) agent's pre-save and post-render gates are wired. Editor save paths pass through pre-save. Render paths pass through post-render. Release is gated on compliance result. Compliance decisions emit Audit Events.

- **M4 — Block Library + Cell Editor UI.** The two surfaces editors use daily. Deep-link support to specific cells. Inline 508(c) feedback. Save path produces new versions. Fan-out editing implemented. Out-of-scope cells render read-only; "Request edit access" path implemented.

- **M5 — Template Builder + Manifest Browser.** Template composition with slot-to-canonical binding. Manifest search and drill-down. Reproduce-from-manifest path.

- **M6 — Activity Log + Role Administration UI.** The Activity & Audit Log surface reads from the Audit Event store with filter, expand-diff, export (PDF + signed JSON), and inline chain-integrity indicator. The Role & Access Administration surface provides user/role/ownership views and the access-request inbox. Every administrative action on this surface itself emits Audit Events.

- **M7 — Deconstruction intake + Fidelity Dashboard.** Proposals from Agent 02 flow into the Approval Queue. The Fidelity Dashboard aggregates grading scores and surfaces regressions.

- **M8 — Stakeholder demo.** Seven PRD scenarios executed live on the corpus.

## 11. Open Design Questions (MVP-critical)

These are questions the engineering team needs to close before or during M1. They are called out here because leaving them latent risks design debt that is expensive to pay later.

- **Variant dimension priority order.** §5.2 references a priority order for tiebreaking. The MVP list is `jurisdiction > line_of_business > channel > audience > plan_type > language > correspondence_sensitivity`, but this should be validated against Governance Lead expectations before M2.

- **Proposed-cell visibility.** Current design: proposed cells are invisible to production renders. This is safe but means a render in a variant that only has a proposed cell (no approved cell and no fallback) fails. Alternative: a "preview render" mode that uses proposed cells. Deferred to M4.

- **Fidelity threshold tuning.** The 95 / 98 thresholds are informed guesses. Calibration against the seed corpus will produce real distributions; thresholds will be locked after M2.

- **Embeddings model selection.** Semantic search for canonical block matching uses pgvector. Model choice (OpenAI text-embedding-3-large vs. an open model) is a cost/latency decision for M1.

- **Manifest storage scale.** Manifests grow indefinitely. The non-negotiable 7-year retention implies a partitioning strategy from day one. Deferred concrete partitioning to M1 implementation but flagged here.

- **Audit log partition strategy.** The Audit Event store will grow faster than the manifest table because it captures every state change, not just renders. Monthly partitioning is the default assumption; 12-month online / 7-year cold is the retention plan; query SLAs across the cold boundary need to be validated against the seed corpus at M1.

- **Scope filter expressiveness.** The MVP supports scope filters over ownership class and variant dimensions. Real-world role definitions may need richer predicates (cross-block regular expressions, effective-date windows). This is intentionally deferred to post-MVP; the MVP open question is confirming the fixed predicate set is sufficient to express every persona's scope in the seed corpus.

- **Service identity root of trust.** Render Consumers authenticate as service accounts. In MVP these are statically provisioned. Post-MVP they federate to the Elevance IdP. The MVP needs a credential-rotation story that does not lose the actor identity in log entries during rotation.

## 12. Relationship to Prior Documents

This tech spec supersedes the AWS/DynamoDB reference in `Project_Argus_Technical_Design.docx` (and the data-model section of `PRD_4_Content_Management_System.docx`) for the CMS data plane only. The compliance agent, deconstruction agent, and rationalization agent specifications remain in force. The TMLCS dialect specification remains authoritative for slot and slug syntax.
