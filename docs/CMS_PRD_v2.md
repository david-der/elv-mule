# Agentic Printing — Content Management System

**Product Requirements Document (v2)**
Elevance Health · Agentic Printing Initiative
April 21, 2026 · MVP Scope · Postgres Reference Implementation

---

## 1. Executive Summary

The CMS is the system of record for every member-facing letter Elevance sends. It is built on a three-layer construction model: a **Structural Template** that defines what a letter looks like, a library of **Managed Copy Blocks** that say what a letter says, and **Substitution Slugs** for member, provider, and claim-specific data that personalize it at render time.

This PRD specifies the MVP build on a relational (PostgreSQL) foundation. It differs from the prior AWS/DynamoDB reference in two material ways.

First, it formalizes **block variants as a first-class concept alongside block versions.** In healthcare correspondence the same structural letter (say, a Medicaid renewal) frequently carries different disclosures depending on the state, program, or plan. A Virginia Medicaid appeal-rights paragraph is not the same copy as the California Medicaid appeal-rights paragraph, and neither should be a "version" of the other — they are siblings. Versioning them as a linear history causes edits in one state to contaminate another. Variants fix this. Every Managed Copy Block is a `(canonical_id, version, variant)` triple, where `variant` is a structured tag describing the jurisdiction, program, line of business, or channel the copy applies to. Selection at render time is deterministic: the variable context supplies the dimensions, the CMS chooses the matching variant, and the audit manifest records exactly which `(version, variant)` pair was used.

Second, it treats **reconstruction fidelity as a measured quantity.** The CMS does not simply render letters — it renders them and then grades them against the original source letter that seeded the template. Every candidate render is compared to its source using a two-track evaluator: a visual track that uses Playwright to screenshot both documents and compute perceptual similarity, and a semantic track that uses the DOCX and PDF skills to compare text content, heading structure, table integrity, and accessibility properties. A letter is "reconstructed" when both tracks score above threshold. This gives Elevance a quantitative basis to claim coverage — not "we built 26,000 templates" but "we have reconstructed 26,000 templates with ≥98% fidelity."

Third, it elevates **ownership, role-based access, and change data capture to first-class MVP requirements.** Enterprise content at Elevance's scale is not a free-for-all; regulatory copy carries legal risk, brand copy carries reputational risk, and every state has an accountable owner. The CMS enforces ownership at the block, variant, and template level; gates every edit, approval, and publish behind role-based permissions; and persists *every* state-changing action in an append-only activity log that is itself immutable and signed. This is the upgrade from today: today there is no durable trail of who changed what, when, why, and against which prior state. Tomorrow, every `(block, version, variant)` transition — and every permission grant, role change, and published render — is reconstructible from the log, forever.

The MVP target: a business editor can edit a block variant (within the scope their role permits), trigger re-render of every dependent letter across every variant combination, and see a graded fidelity report in minutes. No IT ticket. No PHI on disk. Every action — every edit, approval, permission change, and render — persisted in the audit log. Every render auditable back to the exact `(structure, block versions, block variants, context hash, actor, decision chain)` tuple used to produce it.

## 2. Problem & Context

### 2.1 What is broken today

Editing a member letter at Elevance is an IT change request. Business owns the message; IT owns the template artifact. Between them sits weeks of scheduling, approval, and deployment — for a change as small as a regulatory sentence update.

The underlying problem is not tooling. It is architecture. Today's letters are monoliths — structure, copy, and data smeared together inside .docx files that only developers can safely modify. The same appeal-rights paragraph exists in 400 templates. A compliance change to that paragraph is 400 separate edits. Each edit is a risk. Each risk requires QA. Each round of QA takes weeks.

### 2.2 What the three-layer model fixes

The three-layer construction model dissolves the monolith. Structure becomes a small, stable set of skeletons — Elevance expects under 100 at maturity. Copy becomes a governed library of thousands of reusable blocks, each a piece of approved business language that can carry substitution slugs inside it. Data becomes a render-time contract — a variable context passed in at render and never persisted.

*See table in §2.3 for a side-by-side view of the three layers and their ownership.*

### 2.3 Why variants, and why now

| Layer | Role | Owner | Change cadence |
|---|---|---|---|
| **Structural Template** — the skeleton. | Defines what a letter looks like: headers, footers, layout, section order, static headings. | Template Design team (<100 at maturity). | Rare. New versions, not mutations. |
| **Managed Copy Blocks** — the words. | The approved copy library. Reusable blocks that carry substitution slugs inside them. Each block is a grid of variants × versions. | Content Governance + business editors (Compliance, Marketing, Product, Legal). | Frequent. Versioned, approval-gated. |
| **Substitution Slugs** — the person. | Render-time resolution of member, provider, and claim data. | Source systems; resolved per-render, never persisted. | Transient. Per-request. |

The three-layer model as originally specified handles versioning — a block has a stable canonical ID and a monotonic version number; edits create new versions; old versions remain addressable for audit. That works for uniform copy like a corporate greeting.

It does not work for the majority of regulatory copy, which is conditional. Medicare Part D notices in Virginia must include a specific state ombudsman reference that California's notices must not. A prior-auth denial sent to a Medicaid member carries different appeal-rights language than one sent to a commercial member. The same block identifier — say, `appeal_rights_disclosure` — must therefore resolve to different text depending on the letter's dimensions.

Treating these as separate canonical blocks (one `appeal_rights_va_medicaid`, one `appeal_rights_ca_medicaid`, one `appeal_rights_commercial`) causes the library to sprawl into tens of thousands of near-duplicates, and makes cross-variant edits (say, a federal rule change that affects all states) nearly impossible to audit consistently. Treating them as versions of one block is worse — it implies the old version is superseded, when in fact both must remain in force simultaneously.

**Variants is the right abstraction.** A single canonical block carries a matrix of `(version × variant)` entries. A version is a point in time. A variant is a slice of context. Together they form a grid, and the CMS selects a cell from the grid at render time based on the letter's dimensions.

This is the single most important structural decision in this PRD.

## 3. Product Vision

*Be the system of record for structured member correspondence. Govern what appears in every letter. Make block edits a business activity, not an IT ticket. Never store PHI. Prove reconstruction with measurement, not assertion.*

### 3.1 MVP Goals

- Manage Structural Templates, Managed Copy Blocks (with variants and versions), and Variable bindings as three separable, governed assets.
- Support create / read / edit / version / variant operations on blocks, with full lineage and diff across both dimensions.
- Render letters via a synchronous API that takes a structural template ID and a variable context, resolves the correct variant for each block based on the context, and returns .docx, .pdf, and accessible HTML.
- Grade every render against its source letter using the dual-track evaluator (Playwright visual + DOCX/PDF semantic). Persist the score.
- Produce an immutable Render Manifest per render, sufficient for regulatory audit.
- Provide a web UI that lets an authorized business editor find a block, see the full version × variant grid, edit the right cell, and ship without an IT handoff.
- **Enforce role-based access control** across every action — edit, review, approve, publish, render, roll back, administer roles — scoped by block ownership, variant dimension, and template class.
- **Capture every state-changing action** in an append-only, tamper-evident activity log — who, when, what, from-state, to-state, why — and expose it through an Activity & Audit surface that any authorized reviewer can query.

### 3.2 Explicit Non-Goals (MVP)

- No SSO, OIDC, or SAML integration with Elevance IdP. The MVP authenticates against a standalone user directory; production SSO is a post-MVP charter.
- No production-grade PHI handling beyond default encryption and transient variable contexts.
- No print-vendor composition integration (Exstream, FormPrint).
- No digital delivery (email, portal, SMS).
- No multi-region deployment.
- No translation management — Agent 16 handles non-English.
- No brand-voice enforcement — Agent 12 handles that.

## 4. Users & Personas

**Content Governance Lead.** Owns the managed copy library as a whole. Approves structural changes to the variant taxonomy. Reviews rationalization proposals from the Deconstruction Agent. Is the escalation point for conflicts between editors.

**Regulatory Content Owner.** Subject-matter expert for Medicare, Medicaid, HIPAA, and state mandates. Owns the blocks classified as regulatory. Decides when a rule change requires a new version vs. a new variant. The Content Management System must make this decision cheap and explicit.

**Business Editor.** Marketing, Product, and Legal staff. Edits block copy under real-time compliance guardrails (the 508(c) agent runs pre-save). Rarely thinks about variants as a concept — usually they are editing one specific cell (the VA Medicaid disclosure) and need the UI to stay out of the way.

**Template Designer.** Owns Structural Templates. Composes new letter skeletons from existing layouts. Binds slots to canonical block IDs, not to specific variants — variants are resolved at render.

**Render Consumer (system).** Any downstream system that needs a rendered letter: the claims engine, the utilization-management workflow, the enrollment system. Calls `POST /renders` with a template ID and a variable context. Gets back an audit-ready letter.

**Audit Reviewer.** Internal audit, regulatory reviewer, or external auditor. Reconstructs the exact letter a member received and the exact copy variant used.

**Quality Lead.** New persona introduced in this PRD. Owns the grading loop. Reviews fidelity scores across the corpus. Signs off that a template has reached reconstruction threshold. Chases down regressions when a score drops after a block edit.

**Access Administrator.** New persona introduced in this PRD. Owns the role model and the user directory. Grants and revokes access at the role level; assigns block and variant ownership to content owners; investigates activity-log anomalies. The Access Administrator's actions are themselves logged — the administrator is not above the audit trail.

## 5. The Variant × Version Model

### 5.1 What a block actually is

A Managed Copy Block is not a single piece of text. It is a **grid of cells.** The two axes are:

- **Version** — monotonic integer. Increments on every edit. Old versions are never deleted. `v3` replaces `v2` as the current version the moment it is approved.
- **Variant** — a structured tag with named dimensions. Typical dimensions: `line_of_business` (medicare, medicaid, commercial, dual), `jurisdiction` (state code, or `national`), `channel` (print, digital), `audience` (member, provider, broker). A variant is the combination: `{lob: medicaid, jurisdiction: VA}`.

Each cell of the grid — each `(version, variant)` pair — contains one unit of approved copy. Not every cell is populated. A block may have only a `{national}` variant (e.g., a corporate greeting that is the same everywhere). Another block may have a dense variant grid (e.g., appeal-rights disclosures across 50 states × 3 lines of business = 150 cells).

### 5.2 Variant resolution

At render time, the variable context carries the dimensions for the letter being built: the member's state, their line of business, the delivery channel. The CMS resolves each block reference as follows:

1. Take the set of populated variants for that block at the latest approved version.
2. Rank them by specificity against the render context. A variant that matches on all requested dimensions beats one that matches on fewer.
3. Fall back to the `national` or default variant if no specific match exists.
4. If nothing matches and no default exists, the render fails with a specific error — better to fail than to ship wrong copy.

This resolution is deterministic. Given the same block, the same version set, and the same context, the same variant wins every time. The Render Manifest records the winning variant so reproduction is exact.

### 5.3 Why this matters for editors

The practical consequence is that when a business editor opens a block, they do not see a single text field. They see a grid. The UI defaults them into the cell they meant to edit (inferred from filters or direct deep-link). An edit to `{lob: medicaid, jurisdiction: VA}` creates a new version of that cell and leaves every other cell untouched. A federal rule change that affects all Medicaid variants can be authored once and applied as a "variant fan-out" — a single edit that creates new versions in N selected cells.

This replaces today's worst pattern, which is "copy the whole block, change two words, paste it back as a new canonical." That pattern is the reason the library has tens of thousands of duplicates.

## 6. Deconstruction & Reconstruction

### 6.1 Deconstruction — how source letters become templates

The Deconstruction Agent (Agent 02) takes a source .docx or PDF letter and decomposes it into its three layers. Structural elements (headers, footers, layout tables, logos, static section headings) are marked `structural` and extracted into a Structural Template skeleton. Business copy is marked `cms_content` and proposed as Managed Copy Blocks. Member-specific fragments are marked `variable` and converted to substitution slugs in the appropriate namespace.

The existing `Letter_*_DECOMPOSED.html` artifacts in the workspace show the output format — color-coded overlays on the source document, with a corresponding CMS-structure view that shows each extracted block. The CMS ingests these as proposals. A human approves, which promotes the blocks into the canonical library.

### 6.2 Reconstruction — how templates become letters

Reconstruction is the reverse: the Render API takes a Structural Template ID and a variable context, resolves each slot to its correct block variant, substitutes all slugs, and emits .docx / .pdf / accessible HTML. The output is what actually goes to the member.

### 6.3 Fidelity grading — the closed loop

The claim "we reconstructed this letter" has historically been a human judgment call. This PRD makes it a measurement. Every template, on every meaningful change (new version, new variant, structural edit) is rendered with a synthetic variable context that matches the source letter's original data, and the output is compared to the source.

Two evaluators run in parallel.

**Visual (Playwright).** The source letter and the reconstructed letter are both rendered to PDF and then to high-resolution images. Playwright opens both and computes per-page perceptual similarity (pixel-level diff with tolerance for minor font rendering variance). It also renders both as HTML and takes screenshots at representative viewports, comparing layout fidelity. Returns a visual score (0–100).

**Semantic (DOCX/PDF skills).** The PDF and DOCX skills extract text, heading structure, list structure, table structure, hyperlink targets, image alt-text, and document metadata from both documents. The evaluator compares them structurally — same heading hierarchy? same tables with same headers? same alt text? — and textually (with normalization for the variable substitutions). Returns a semantic score (0–100) with a per-category breakdown.

A template is considered "reconstructed" when visual ≥ 95 and semantic ≥ 98. Below those thresholds, the template stays in `calibration` state and is routed to the Quality Lead's queue with a diff view showing exactly what differs.

This gives the program a defensible coverage number. "We have 1,247 templates at reconstruction threshold" is a claim Elevance can put in front of CMS auditors.

## 7. Ownership, Access, and the Audit Trail

This section is the other structural claim of the v2 PRD, on par with variants. Where §5 says *the content is a grid*, this section says *every cell of that grid has an owner, a permission surface, and a full history of every action ever taken against it*. An enterprise content platform without those three things is a shared Google Doc.

### 7.1 Ownership — who is accountable for each asset

Ownership is explicit, recorded in the data model, and surfaced in the UI. There is no orphaned content.

**Block ownership.** Every canonical Managed Copy Block has a designated owning role (`regulatory`, `marketing`, `product`, `legal`, `brand`, `clinical`). The role is held by a named person or group. The owner is the single accountable party for the copy of that block across all its variants and versions. Ownership can be delegated per variant — e.g., the `appeal_rights_disclosure` block is owned by `regulatory`, but the `{lob: medicare}` variants are stewarded by the Medicare sub-team and the `{lob: medicaid}` variants by the Medicaid sub-team.

**Template ownership.** Every Structural Template has an owning team (typically the Template Design team). Ownership carries the responsibility for slot shape, not copy. When a template is deprecated, the owner is the person who signs the deprecation.

**Manifest ownership.** Render Manifests are owned by the render requester (the system that called `POST /renders`) and co-owned by the CMS for audit purposes. Manifests are append-only and cannot be rewritten.

**Unowned content is invalid.** Any asset without a valid current owner fails validation and enters a quarantined state. The Access Administrator is alerted. This prevents the state where an editor leaves the company and their blocks quietly drift into nobody-can-edit-this limbo.

### 7.2 Role-based access control

The MVP ships with a fixed set of roles, each with an explicit permission scope. Roles compose additively — a user can hold multiple roles, and their effective permissions are the union.

| Role | Read | Edit cell | Submit for approval | Approve | Publish | Roll back | Admin |
|---|---|---|---|---|---|---|---|
| Viewer | ✓ | | | | | | |
| Business Editor | ✓ | ✓ (in scope) | ✓ | | | | |
| Regulatory Content Owner | ✓ | ✓ (regulatory) | ✓ | ✓ (regulatory) | | | |
| Content Governance Lead | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | |
| Template Designer | ✓ | ✓ (templates) | ✓ (templates) | | | | |
| Quality Lead | ✓ | | | ✓ (fidelity) | | | |
| Render Consumer (system) | ✓ (by manifest) | | | | ✓ (render-only) | | |
| Audit Reviewer | ✓ (manifests + log) | | | | | | |
| Access Administrator | ✓ | | | | | | ✓ |

Edit scope is enforced along two axes: the **block's owning role** (a Business Editor tagged `marketing` cannot edit a `regulatory` block), and the **variant dimensions** (a Regulatory Content Owner can be scoped to Medicare only). When an editor opens a cell they lack permission to edit, the UI renders it read-only with an explicit "Request edit access" action routed to the Access Administrator.

Every permission check is recorded — both grants and denials. A denial with the wrong role still produces a log entry, which is what allows the security team to notice attempted privilege escalation.

### 7.3 The activity log — change data capture as the system's memory

Every state-changing action in the CMS produces exactly one entry in the activity log. The log is the durable, append-only, tamper-evident record of everything the CMS has ever done. It is the single source of truth for audit.

**What is captured.** The following actions always produce a log entry, without exception:

- Block create, cell create, cell edit, cell version promotion, cell rollback, cell retirement.
- Variant dimension add / retire. Variant population change.
- Template create, slot bind / unbind, template publish, template deprecate.
- Approval request submit, approve, reject, comment.
- Render execute, render fail, render held for compliance review, fidelity score record.
- Permission grant, permission revoke, role assignment change, ownership transfer.
- Rationalization proposal intake, accept, reject.

**What each entry records.** The log envelope is stable across every action. Every entry carries: actor identity, actor's effective role set at time of action, timestamp (UTC, millisecond-precise, monotonic), the target entity (type + canonical ID + version/variant when applicable), the action type, the from-state and to-state (for state transitions), the payload diff (for edits, a structured diff of what changed, with PHI scrubbed), the reason code / free-text justification, a correlation ID linking related entries (all actions in a single governance flow share one), and a cryptographic hash of the entry chained to the previous entry's hash. That chain is what makes the log tamper-evident — altering a historical entry breaks every subsequent hash.

**What the log never records.** The actual variable context of a render (that stays transient per the PHI-zero-durability rule — only its hash goes into the manifest and the log). No PII beyond actor identity. No free-text fields that have not been sanitized.

**Retention.** The MVP keeps activity log entries indefinitely. There is no rolling delete. Logs older than 7 years are archived to cold storage but remain queryable with a latency SLA.

**Access.** The Activity Log surface is readable by any role with the `audit_read` permission (all governance and audit roles). Write access is restricted to the CMS itself; no human or role can write a log entry directly, and no entry is ever modified after write.

### 7.4 Why this is the right moment

Today, content changes at Elevance are invisible to audit. A compliance change goes through IT ticketing, Jira comments, Confluence notes, email chains, and eventually a production deploy — and the final letter has no durable link back to the business decision that drove it. When a regulator asks "why did this disclosure change in April," the answer is reconstructed from human memory and ticket archaeology.

The CMS is being built at the same moment Elevance is consolidating member correspondence onto a governed platform. If CDC is not first-class on day one, it will never be retrofit cleanly — audit trails are notoriously hard to add after the fact because they depend on every action in the system emitting its events correctly from the start. Building it in now is the only way it becomes reliable, and the only way it becomes a feature editors actually trust.

This gives Elevance three things at once: defensible regulatory posture (every action explainable to CMS, state departments, internal audit), internal operational hygiene (who is actually doing the work, where are bottlenecks, which blocks see the most churn), and a foundation for future governance automations (the 508(c) agent, the brand-voice agent, and the rationalization agent all consume the activity log as an input).

## 8. User Experience Principles

The CMS is used by people who are not developers and do not want to learn a markup language. Four principles shape every screen.

**Show the grid, not the graph.** Business editors think in "the VA Medicaid disclosure," not in canonical IDs and version graphs. The primary edit surface is a spreadsheet-like grid of variants × versions, with the editor landing in a single cell. Advanced users can pop the lineage/graph view open on demand; 90% of edits never need it.

**Make the right action the easy action.** If an editor wants to change the Medicaid appeal-rights paragraph for Virginia only, the default save behavior creates a new version of that variant only — not a fan-out. Fan-out is an explicit, separate action, so "I changed one state" never accidentally becomes "I changed all fifty."

**Diff everything.** Every save shows a before/after diff. Every approval request shows what exactly changed and which variants it applies to. Every render that fails fidelity shows the semantic and visual diff inline. Reviewers never approve blind.

**Never leave the editor in an invalid state.** Unresolved variables, broken slugs, missing required variants, 508(c) violations — all surfaced as inline indicators while typing. The editor cannot save a block that would fail to render. This shifts failures left, where they cost minutes, not release cycles.

## 9. User Flows (MVP)

### Flow A — Editor updates a block variant

*Sarah, a Regulatory Content Owner, needs to update the Virginia Medicaid appeal-rights disclosure in response to a state rule change effective next month.*

1. Sarah opens the CMS. The home screen is the **Block Library** — a searchable, filterable list of every canonical Managed Copy Block.
2. She searches `appeal rights medicaid`. The library returns the canonical block `appeal_rights_disclosure`, with a preview of the variant matrix (columns: jurisdictions; rows: lines of business). A small badge shows which cells have been updated recently and which have open review requests.
3. She clicks into the block. The **Block Editor** opens on the grid view. Each cell shows the current version number and a snippet of the copy. She clicks the VA × Medicaid cell.
4. The editor opens that cell. Left pane: the variant dimensions (VA, Medicaid), the current version (v4), the list of letters using this variant, and a fidelity indicator (green if current renders all pass grading). Center pane: the rich-text body of the block, with substitution slugs rendered as inline chips. Right pane: inline 508(c) guardrail feedback.
5. Sarah edits the paragraph. As she types, a chip for `{{member.state_name}}` remains stable, the 508(c) panel clears, and a "Save as v5" button activates.
6. She clicks save. A confirmation modal shows: "This will create version 5 of the VA × Medicaid variant. It will NOT affect other states or lines of business." She confirms.
7. The block enters `proposed` state. The 508(c) agent re-scans, returns green. A notification goes to the Content Governance Lead's approval queue.
8. The Governance Lead approves. The new cell promotes to `approved`. Every Structural Template that uses `appeal_rights_disclosure` is now queued for re-render with the new variant where applicable; dependents are listed inline.
9. Sarah receives a report: "47 letters re-rendered. 47/47 pass grading. Zero fidelity regressions."

### Flow B — Editor fans out a change across multiple variants

*A federal rule change affects all Medicaid variants, every state.*

1. Sarah opens the same block. On the grid view, she selects the entire Medicaid row.
2. She clicks **Edit selection**. The editor opens a special fan-out mode. She types the new paragraph once. The preview panel shows how it will look in each of the 50 selected cells (substitution slugs resolved against each state's sample context).
3. She saves. A confirmation shows every cell that will be versioned. She confirms.
4. 50 new versions are created — one per state. All enter `proposed` state together and share a single approval request.
5. The Governance Lead approves once. All 50 cells promote together. The Render Manifest for any affected letter will record the specific state's cell.

### Flow C — Template designer adds a new letter type

*A new correspondence type — "Preventive Care Gap Reminder" — needs a template.*

1. Terrell, a Template Designer, opens the **Template Library** and clicks **New Template**.
2. He selects a base layout — "Standard Member Notice." The new template is created as a draft.
3. The Template Builder opens. It is a slot-ordered view: header, greeting, body, disclosures, closing, signature. Each slot shows its type (greeting, body, disclosure, etc.) and is currently empty.
4. For each slot, he searches for a canonical block. The search is semantic — typing "welcome" returns `welcome_body_general`. He drags it into the greeting slot. The slot binds to the canonical ID only; variants resolve at render.
5. For the disclosures slot, he binds `preventive_care_reminder_disclosure`. Its variant matrix becomes part of the template's dependency list.
6. He hits **Preview**. A modal asks for sample context (or picks synthetic defaults). The render pipeline produces a .pdf preview using the default variant cells.
7. When the template looks right, he **submits for approval**. The Governance Lead approves. The template moves to published. It is now callable via the Render API.

### Flow D — Render Consumer calls the Render API

*The utilization-management system needs to send a prior-auth decision letter to a member.*

1. UM posts to `/renders` with `{ template_id: "prior_auth_decision", variable_context: { member: {...}, auth: {...}, ... } }`.
2. The CMS resolves each slot's bound canonical to the correct `(version, variant)` cell using the variable context's `line_of_business`, `member.state`, and `channel` dimensions.
3. Every slug is substituted. The output is rendered to .docx, .pdf, and HTML in parallel.
4. The 508(c) post-render scan runs. If pass, the outputs are returned. If fail, the render is held and a governance alert is emitted.
5. The Render Manifest is written to the audit table: template ID + version, each slot's resolved `(canonical, version, variant)`, variable context hash, output hashes, compliance scan ID, fidelity score, requesting system identity.
6. The activity log records a `render.succeeded` entry linked to the manifest (or `render.held_for_compliance` if the scan failed), with the requesting system as actor.
7. UM receives the letter. The variable context is discarded. Only the hash remains.

### Flow E — Audit reviewer reconstructs a member's letter

*A regulator asks: "What did we send this member on April 3?"*

1. The Audit Reviewer opens the **Manifest Browser** and searches by date + member policy number. The system returns the Render Manifest.
2. The manifest shows: template X, version 7; 14 slots; each slot's resolved block canonical + version + variant; variable context hash; output hash; compliance pass.
3. The reviewer clicks **Reconstruct**. They supply the original variable context (retrieved from the source system). The CMS renders again. The output hash matches. The letter is identical to what the member received.
4. The reviewer clicks through to the **Activity Log** view for the template and each resolved block. The log shows the full history — when each cell was last edited, who approved it, and which prior versions were superseded.

### Flow F — Access Administrator grants scoped access and reviews activity

*A new Medicaid compliance hire, Priya, needs edit access to Medicaid-classified blocks only, and the Access Administrator needs to verify she is working within scope a week later.*

1. The Access Administrator opens the **Role Administration** surface. They search for Priya's user record, click **Assign role**, and select `Regulatory Content Owner` with a scope filter of `{lob: medicaid}`.
2. A justification field is required. The administrator types "Onboarding per HR ticket #4812." They confirm.
3. The assignment is persisted. The activity log records a permission-grant entry — actor, target user, role granted, scope, justification, correlation ID for the onboarding flow.
4. Priya logs in. The Block Library shows only blocks she is authorized to view in full; the grid surfaces lock the cells outside her scope.
5. A week later, the Access Administrator opens the **Activity & Audit Log** surface, filters by Priya's user ID over the last 7 days, and reviews her edits, approval submissions, and any denied actions. Every row is expandable to show the structured diff and the before/after state.
6. Nothing out of scope appears. The administrator exports the week's log entries as an audit record (PDF + signed JSON) and attaches it to the compliance review ticket.

## 10. Screen Inventory (MVP)

The UX is organized around nine primary surfaces. Each is designed to be usable without training.

**1. Home / Block Library.** A searchable, filterable list of every canonical Managed Copy Block. Filters across block type, domain, regulatory class, variant dimensions, approval status. Each row shows the block name, a copy preview, its variant footprint (a small grid thumbnail), and the number of templates that depend on it. This is where most editors start their day.

**2. Block Editor (Grid View).** The primary editing surface for a block. The grid shows variant dimensions as rows/columns (configurable axes). Each cell shows the current version and a copy snippet. Color indicates status: green approved, yellow proposed, gray draft, red fidelity regression. Click a cell to open the Cell Editor. Multi-select cells to enable fan-out.

**3. Block Editor (Cell Editor).** The focused editing surface for a single `(variant, version)` cell. Rich-text editor with substitution slugs rendered as inline chips and a live namespace-validated completions dropdown. Sidebar with: variant tags, version history (with diff against previous), dependents list, live 508(c) guardrail feedback. A preview panel renders the cell with sample context alongside.

**4. Template Library.** Parallel to the Block Library but for Structural Templates. Each row shows the template name, correspondence type, last-published version, and a traffic-light fidelity indicator (aggregated from the templates' recent renders).

**5. Template Builder.** A slot-ordered view of a template. Drag-and-drop slot reordering, slot-type constraints, semantic search for canonical blocks to bind into each slot. Preview with sample context. Validation panel showing any missing required slots, broken bindings, or unresolved dependencies.

**6. Approval & Governance Queue.** A single inbox for every pending action — proposed block edits, proposed templates, rationalization proposals from the Deconstruction Agent, fidelity regressions that need triage. Bulk actions (approve REUSE proposals, dismiss noise). Each item shows a diff view so reviewers approve with full information.

**7. Manifest Browser.** The audit-path surface. Search Render Manifests by date, template, member (by hash), status. Drill into a manifest to see every block and variant used, reproduce the render with supplied context, and export the audit record. This is primarily for Audit Reviewers and the Quality Lead.

**8. Activity & Audit Log.** The full CDC surface. A chronological, filterable ledger of every state-changing action the CMS has ever recorded. Filters across actor, role, action type, target entity (block / variant / version / template / manifest / permission), date range, and correlation ID. Each row is expandable to show the structured diff between from-state and to-state, the justification, and the hash chain neighbors. Export as PDF audit record or signed JSON bundle. This is where the Audit Reviewer, Content Governance Lead, and Access Administrator triangulate any question about what actually happened.

**9. Role & Access Administration.** The Access Administrator's home. A user directory view (who is in the system), a role view (what the current role model is and who holds each role), and an ownership view (who owns each block and template). Every change here requires a justification and emits an entry to the activity log. A separate "Access Requests" inbox receives requests from editors who are blocked by their current scope; the administrator can approve, deny, or route them to the Content Governance Lead.

A dedicated **Fidelity Dashboard** aggregates grading scores across the corpus — visible from a nav link in the top bar of every screen. It surfaces recent regressions, corpus coverage (% of templates at reconstruction threshold), and trend lines.

## 11. MVP Scope

### 10.1 In

Block Library, Block Editor (grid + cell), Template Library, Template Builder, Approval Queue, Manifest Browser, Activity & Audit Log, Role & Access Administration, Fidelity Dashboard. Postgres-backed data layer. Sync render API. Dual-track grading evaluator. 508(c) pre-save and post-render gates. Rationalization intake from the Deconstruction Agent. Render Manifests with hash-based reconstructibility. Role-based access control with scope filters on block ownership and variant dimensions. Append-only, hash-chained activity log capturing every state-changing action. Ownership assignment and validation (blocks, variants, templates).

### 10.2 Out

SSO / OIDC / SAML integration with Elevance IdP. Translation. Brand-voice enforcement. Print-vendor integration. Digital delivery. Async batch renders at scale (a single-batch mode is in; the production batch throughput target is out). Cost attribution. Multi-region. Fine-grained attribute-based access policies beyond the fixed role set shipped in MVP.

### 10.3 Explicit MVP corpus

75 seed letters from G&A expanding to 500+ letters across G&A and Utilization Management. Seven variant dimensions in scope: `line_of_business`, `jurisdiction`, `channel`, `audience`, `plan_type`, `language` (single-language MVP; the field exists but only `en` is populated), `correspondence_sensitivity`.

### 10.4 Success Criteria

- **Reconstruction coverage.** 100% of the 75 seed letters reach reconstruction threshold (visual ≥ 95, semantic ≥ 98).
- **Variant correctness.** 100% of render manifests record the correct `(version, variant)` pair for each slot, verified against known-good test contexts covering every populated variant cell.
- **Edit latency.** A business editor can open a specific cell, edit, submit for approval, have the 508(c) scan pass, and see the re-render grading report complete in under 90 seconds on the MVP corpus.
- **Propagation correctness.** Editing one cell re-renders exactly the letters that use it, and no others. Zero false positives, zero false negatives on the corpus.
- **PHI zero durability.** No variable context values in any durable store. Only hashes. Audited by scan.
- **Audit reconstructibility.** 100% of past manifests in the corpus re-render byte-identically given the original context.
- **Zero IT tickets.** In the MVP test period, zero content changes required an IT ticket. All edits went through the governance UI.
- **Access enforcement.** 100% of out-of-scope actions attempted during the test period are blocked with a logged denial — no out-of-scope writes succeed. Zero privilege escalations via the API.
- **Log completeness.** 100% of state-changing actions during the test period appear in the activity log, verified by an independent reconciliation pass over the database state.
- **Log integrity.** A randomly sampled set of 1,000 log entries is verified to chain correctly (hash of each entry matches the recorded hash of the next). Zero tampering detectable or undetected.
- **Ownership coverage.** 100% of canonical blocks and structural templates have a valid current owner at all times during the test period. Zero orphans.

### 10.5 Demonstration scenarios

1. **End-to-end reconstruction.** A legacy .docx enters the Deconstruction Agent; the CMS ingests the proposals; the Governance Lead approves; a render passes grading with a green fidelity score.
2. **Single-cell edit.** A Regulatory Content Owner edits the VA × Medicaid appeal-rights cell; exactly the letters using that cell re-render; every other variant is untouched.
3. **Fan-out.** A federal rule change is applied across all 50 Medicaid state cells in one authoring action; all 50 new versions promote together on a single approval.
4. **Fidelity regression catch.** A deliberately introduced layout break in a Structural Template causes grading scores to drop; the Quality Lead sees the regression flagged in the dashboard within the render cycle.
5. **Audit recall.** A date + member policy lookup returns the Render Manifest; re-rendering with the original context produces a byte-identical letter.
6. **Scoped access enforcement.** An editor with `Regulatory Content Owner — Medicare` attempts to edit a Medicaid block; the UI blocks the action, the API denies the write, and both the attempt and the denial appear as a single correlated entry in the activity log within two seconds.
7. **Activity log reconstruction.** The activity log is queried for a single block across a six-month span; the resulting timeline reconstructs every cell, version, approval, and publish event, and the current block state can be rebuilt from the log alone with no reference to the primary data tables.

## 12. Risks

**Variant taxonomy sprawl.** If the variant dimensions are not disciplined, editors will create ad hoc dimensions and the selection algorithm becomes unpredictable. Mitigation: the variant dimension set is governed — new dimensions require Governance Lead approval. The MVP ships with a closed set of seven.

**Fidelity grading false negatives.** Playwright visual diff can be fooled by trivial font rendering differences; semantic diff can be fooled by acceptable phrasing changes. Mitigation: thresholds are conservative (95 / 98) rather than perfect; the Quality Lead reviews anything below threshold; the evaluator's weights are tuned against the seed corpus.

**Propagation blast radius.** A change that accidentally fans out wider than intended could re-render thousands of letters incorrectly. Mitigation: the UI requires explicit confirmation of fan-out scope; every re-render goes through grading before it is released; anything below threshold is held, not published.

**PHI leakage through logs.** A well-intentioned developer logs the variable context and spills PHI into CloudWatch. Mitigation: structured log schema prohibits the context field; static analysis on the logging pipeline; periodic audit.

**Postgres scale ceiling.** Hot-path reads on the block catalog could become a contention point at high render volume. Mitigation: the MVP is single-region, single-writer, with a read replica; a Redis cache fronts the block resolution path; production scale-out planned in the post-MVP charter.

**Activity log write amplification.** Every action produces at least one log entry, sometimes several (a bulk fan-out creates N cell events plus one parent event). At steady state the log table will grow faster than the catalog. Mitigation: the log is partitioned by month; cold partitions are offloaded to archival storage on a rolling 12-month window; query paths use a materialized index by entity + time so read latency stays flat as the log grows.

**Role model rigidity.** A fixed set of MVP roles cannot cover every permission permutation a real enterprise content org will eventually want. Mitigation: the MVP ships with the roles that cover the seed corpus and explicitly-identified personas; a "request edit access" path lets editors escalate gaps into the Access Administrator's queue; the post-MVP charter plans for attribute-based access control once the role model has been exercised in production.

**Log pollution by non-actionable events.** If every trivial click emits an entry, the log becomes noise and reviewers stop reading it. Mitigation: the scope of captured events is explicitly enumerated in §7.3; read actions and UI navigation do *not* emit log entries; only state-changing actions do. This keeps signal density high.

## 13. MVP Milestones

| Milestone | Week | Deliverable |
|---|---|---|
| M1 — Postgres schema & core CRUD | End W2 | Blocks (with variants), Templates, Manifests, Activity Log, Users/Roles/Ownership; create/read/update/version/variant endpoints with permission checks wired in |
| M2 — Render pipeline + grading evaluator | End W3 | Sync render API; Playwright + DOCX/PDF graders; fidelity score persistence; render events emitted to activity log |
| M3 — Compliance integration | End W4 | 508(c) pre-save and post-render gates live; compliance events logged |
| M4 — Governance UI (Block Library + Cell Editor) | End W5 | The two surfaces editors use daily, with scoped views based on role |
| M5 — Template Builder + Manifest Browser | End W6 | Designer and Auditor surfaces |
| M6 — Activity Log + Role Admin UI | End W6 | The audit surface and the administration surface live, with export and hash-chain verification |
| M7 — Deconstruction intake + Fidelity Dashboard | End W7 | Proposals from Agent 02; dashboard of corpus fidelity |
| M8 — Stakeholder demo | W7 | All seven demonstration scenarios run live on the corpus |

## 14. Appendix A — Relationship to Prior Documents

This PRD supersedes `PRD_4_Content_Management_System.docx` (DynamoDB-reference, April 14) in two material ways: the storage tier moves to PostgreSQL for relational integrity of the variant grid, and variants are elevated from an unspecified future concern to a first-class MVP concept with its own data model and UX. Functional requirements in the prior PRD that do not conflict with this document remain in force; where they conflict, this PRD governs.

Deconstruction specifics live in `PRD_1_Template_Deconstruction_Agent.docx`. Rationalization specifics live in `PRD_2_Template_Rationalization_Agent.docx`. 508(c) specifics live in `PRD_3_508C_Compliance_Agent.docx`. The TMLCS dialect specification (`TMLCS_Dialect_Specification_v1.docx`) remains the authoritative source for slot and slug syntax.

## 15. Appendix B — Companion Documents

- **Technical Specification** (`CMS_Tech_Spec_v2.md` / `.docx`) — the conceptual Postgres data model, pipelines, and integration contracts.
- **UX Prototype** (`CMS_Prototype.html`) — a clean, interactive HTML mock-up of the Block Library and Cell Editor surfaces so you can feel the interaction before engineering builds.
