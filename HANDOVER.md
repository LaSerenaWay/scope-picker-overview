# Scope Picker — Handover (updated 2026-06-05, session 2)

Continuation of the BlueDolphin E2E Solution Navigator work. The biz-arch model in **Import Test 01** is built and verified. This handover supersedes the prior one and covers the Scope Picker as it stands after the Step 3 rework, the new-object authoring feature, and the apply_scope.py requirement load.

---

## 1. What the Scope Picker is

A standalone POC web tool (single self-contained `.html` artifact) that lets a solution architect take the **master/truth model** and carve out a **project-specific scope**, including authoring objects that don't exist in master yet.

- **Standalone for now**, designed to later drop into the Navigator as a new tab.
- **Fully file-driven** (no live BD calls). Migrate to live BD later.
  - **Input:** extended `e2e_data.json` (with `reference_data`) + a requirements file (`.xlsx`/`.csv`).
  - **Output:** `scope_instruction.json` -> consumed by `apply_scope.py`.

### Three-step flow
1. **Project Setup** — pick applicable Product Categories, Customer Segments, Channels, Application Components (4 shortlists). Each pick card has a **+ New** inline input to add a non-master reference item (title only; auto-selected; pink NEW badge).
2. **Requirements Mapping** — upload requirements -> editable grid. Per requirement (via drawer): tag Channel/Product/Segment + App Component(s), select one or more E2E Use Cases, and under each E2E tick its Business Services (core vs conditional). Bulk apply + save/resume supported. Authoring of new E2E/BS/BP happens here (see §2) so requirement context is never lost.
3. **Review & Generate** — three sub-tabs: **Scope Navigator**, **Coverage Stats**, **Generate**.

---

## 2. Step 3 — current state (this is where most session-2 work went)

### Sub-tab: Scope Navigator
- Read-only walk of the scoped model, filtered to objects the requirements actually selected.
- Left pane: **Journey is the default grouping** (Value Stream toggle available). Default E2E selection = **first** scoped E2E in display order.
- Right pane: selected E2E -> core/conditional **Business Service** cards. Each card is collapsible inline:
  - **Business Process** variants shown inline (bounded set, ~5-6). Each BP row expandable to show applicable Channel/Product/Segment, or base-flow note.
  - **Requirements** open in a right-side **drawer** (not inline — avoids long scroll). Drawer lists each requirement's id + full statement, plus that requirement's **other mappings as chips (Channel/Product/Segment/App Component, NOT E2E)**.
- CRITICAL FIX: scope is tracked **per-E2E** (`bsByE2e` / `bpByE2e` in `computeScope()`). A conditional BS picked under one E2E no longer leaks into other E2Es that list it.

### Sub-tab: Coverage Stats
- **Two-column grid** (`.cov-grid`, `align-items:stretch` so cards fill row height). Cards: 4 dimension bar charts + Grouping coverage + Gaps.
- Bars chart **only the Step 1 shortlist** for each dimension (not full master). Empty-shortlist dimensions show a prompt.
- Bars AND (the now-hidden) shortlist chips are **clickable** -> `openDimDrawer()` shows the E2Es + Requirements tied to that item.
- **Project shortlist card is HIDDEN** — `host.appendChild(shortlistCard())` is commented out in `renderCoverageStats()`; `shortlistCard()` fn kept intact to unhide later.
- Custom lightweight bars (no chart lib) — chosen for zero CDN dependency + easy click wiring. ECharts is the agreed upgrade path if richer visuals wanted.

### Sub-tab: Generate
- Validation block (errors gate the button) + scorecards + Project context review + JSON preview/download. Unchanged logic.

---

## 3. New-object authoring (session 2 feature)

Lets the architect add objects NOT in master, then export so the applier clones master ones and creates new ones.

- **Reference data**: + New inline on each Step 1 pick card. Exports as `{id, title}` only.
- **New E2E**: in the Step 2 drawer, **+ Create new E2E Use Case** -> full BOEM form (Title, JTBD, Business Outcome, Scope Owns, Scope Doesn't Own, Description) + Value Stream + Journey dropdowns.
- **Business Services on an E2E** (new E2E starts empty, so both paths exist):
  - **+ Attach existing Business Service** — searchable multi-select of master BS not already on the E2E, with Core/Conditional selector. (New E2E != new BS; an E2E can be assembled mostly from existing BS.)
  - **+ Add new Business Service** — full BOEM form + Core/Conditional + condition text.
- **Business Process variant**: **+ Business Process variant** button (NO prompt() — inline, iframe-safe). The variant **implicitly inherits the requirement's selected Channel/Product/Segment** from the top of the drawer (shown as chips for transparency). Rule (independent of new vs reuse BS): a brand-new BS's FIRST BP = **base flow** (no context); every other BP added anywhere = **variant** carrying the inherited context tags.
- Temp ids: `new-<kind>-<n>` via `newId()`; `S.newSeq` counter. `isNew(o)` checks `origin==='new'`.

### Key gotchas fixed this session (do not regress)
- **Wrong core/conditional on save**: `mapSummary()` must derive flavour from **list membership** (is the BS in the E2E's `conditional_services`?), NOT the mutable `_flavour` flag — the flag was stale after a new core BS round-tripped and showed as conditional.
- **No mid-edit refresh**: removed `persistDraftLive()`. Save is the single commit point; creating/attaching BS or adding BP only rebuilds the drawer, then **Save mapping** commits to `S.map` and refreshes grid + review.
- **Labels**: visible UI says "Business Service"/"Business Process", not BS/BP.
- **Two-col card stretch**: `.cov-grid { align-items:stretch }` (was `start`, left short cards not filling height).

---

## 4. scope_instruction.json (schema scope_instruction/v1) — current shape

- `project{name,prefix,replaces_prefix,materialize}`, `source_workspace`, `project_context{...}`, `scope{e2e_use_cases,business_services,business_process_ids,app_service_ids}`, `requirements[]`.
- Every packed object carries **`origin: "master" | "new"`**.
- **`new_objects`** block (top level): `reference_data{product_categories,customer_segments,channels,application_components}` (id/title), `e2e_use_cases` (id, title, boem, value_stream_id, journey_id), `business_services` (id, e2e_id, title, flavour, remark, boem), `business_processes` (id, business_service_id, title, **kind: base|variant**, applicable_channels/products/segments).
- Variant context ids may themselves be new (e.g. a just-added channel) -> applier resolves via id-map.

---

## 5. apply_scope.py — DONE this session (drafted+extended, still never run live)

Materializes the instruction into BD. `origin master` -> CLONE into `[Project]`-prefixed copy (BOEM carried). `origin new` -> CREATE fresh. Relationships recreated **between the new copies only** (never back to [MASTER]).

- Objects first (E2E -> BS -> BP -> new reference data), then relationships, then requirements.
- **BP variants don't exist in BD yet** — applier creates them: Realization BP->BS, and for variants, **Association BP -> each applicable Channel/Product/Segment** (records applicability). Base BP gets Realization only.
- **Requirements** (§ step 6): each in-scope requirement -> a BD **"Requirement"** object (type discovered by NAME via `discover_type_by_name("Requirement")` since no master instance exists). Linked **E2E -> Requirement** and **BS -> Requirement** via **Realization** (element realizes requirement). No BP/Channel/etc links from requirement.
- `object_type_id` discovered at runtime off an existing object (never the OData definition id — BD rejects that on POST).
- Idempotent (`[Project]`+title reuse), has `--dry-run`. `new_workspace` mode stops with guidance.
- **Always `--dry-run` first.** Never executed live yet. First run: confirm the "Requirement" definition name matches in `GET /v1/object-definitions`.

---

## 6. BD API quick-reference (confirmed, carried forward)

- **REST:** base `https://public-api.eu.bluedolphin.app/v1`; headers `{x-api-key: 31243ea1-19db-48fe-9eba-821eb71bad55, tenant: csgipoc, Content-Type: application/json}`. POST objects: `object_title`, `object_type_id` (=definition id read off existing object), `workspace_id`. Rename via PATCH `object_title`. BOEM via PATCH `boem:[{id,name,items:[{id,name,value}]}]`. Relations: POST `from_object_id/to_object_id/template_id`; condition via PATCH `remark`.
- **OData:** base `https://csgipoc.odata.bluedolphin.app`; Basic `base64(csgipoc:ef498b94-732b-46c8-a24c-65fbd27c1482)`. Objects filter `Workspace eq 'Import Test 01' and Status ne 'Archived'`. Relations filter `BlueDolphinObjectWorkspaceName eq 'Import Test 01'`.
- **Templates:** Association `686b64bda20b34f4325a6884`, Aggregation `686b64bba20b34f4325a6874`, Realization `686b64bba20b34f4325a6876`, Composition `686b64bba20b34f4325a6872`.
- **BOEM (Business Service):** section "Ameff properties" `68951b65d14a12a209a5cb42` — JTBD `d0d74cd7-1098-45c3-b46c-304cff725801`, Business Outcome `1b661cd4-b900-4612-a0f4-ca70e61a2af5`, Scope Owns `5f74113d-c7f3-4380-b7f4-c6dd5571188a`, Scope Doesn't Own `57c53c17-1ddc-40d3-9315-004913781ee8`. Description: section "Convention Model" `69dd0b985c7871c42929884b`, field `b8c7f74c-53d6-440b-b2aa-b94fd6ae3b88`.
- **Workspace:** Import Test 01 = `69dfb4b8416b82addb78025d`. Pulak user_id `68627da6fd3ef99d04e7c5f0`.

---

## 7. Files (current locations)

- **scope_picker.html** — `/home/user/files/scope_picker.html` (latest, on canvas, validated `node --check`). The working artifact.
- **apply_scope.py** — `/home/user/files/apply_scope.py` (latest, on canvas).
- **extract_e2e_data_v3.py** — `/home/user/files/extract_e2e_data_v3.py` (OData-only extractor with reference_data + TMF filter).
- **e2e_data.json** — `/home/user/e2e_data.json` (extractor output; what-vs-how model + reference_data).
- **Requirements.xlsx** — `/home/user/Requirements.xlsx` (header row 2: Type/Compliance/UID/Statement; ~1,463 rows; Statement filled, UID empty).
- **e2e_navigator.html** — `/home/user/files/e2e_navigator.html` (the standalone Navigator, reference for the embedded Scope Navigator view).
- INDEX.md — `/home/user/files/INDEX.md`.

---

## 8. Working preferences (carry forward)

Ask, don't assume. Pragmatic/scope-conscious; reuse patterns, avoid over-engineering. Human, simple language. Render Python in markdown code blocks (no sandbox `.py` download). **Surgical `artifact_edit` over full rewrites for HTML** — full rewrites have broken the working file before; edit disk copy + validate, then sync canvas. One thing at a time. When low on budget, deliver validated downloads and defer the canvas push.

---

## 9. Open threads / next steps

1. **Run apply_scope.py `--dry-run`** against a small scope, confirm "Requirement" type name match, then a single-requirement live test before full apply.
2. Optionally fold App Components into Save/Load progress and the Step 3 review block.
3. ECharts upgrade for Coverage Stats bars if richer interactivity wanted.
4. App Service filtering (App Component -> App Services per BS) — deferred until BS->App Svc links exist in BD.
5. + Business Process variant currently uses inline form (good); confirm behaviour once BD BP-variant model exists.
6. Consider unhiding the Project shortlist card if it becomes useful again (one-line uncomment in `renderCoverageStats`).
