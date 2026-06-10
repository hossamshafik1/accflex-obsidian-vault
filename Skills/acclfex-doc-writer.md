`---`

`name: acclfex-doc-writer`

`description: "Use this skill whenever the user wants to document an Acclfex ERP module, screen, aggregate, or feature — for both business and technical audiences. Triggers include: 'write docs for', or any request to produce structured documentation from existing code and/or an ADO work item. Also use when the user mentions a screen name, aggregate, or feature and wants Claude to read the code and produce a rich explanation of what it does. Use this even if the user says 'just explain' or 'summarize' a module — if reading code and producing documentation is involved, this skill applies. Covers all Acclfex repos: Angular frontend, C# backend APIs, SQL, WinForms desktop. Output is always written to the Obsidian vault via obsidian-mcp."`

`---`

`# Acclfex Documentation Writer`

`Produces rich, layered documentation for Acclfex ERP modules by reading existing`

`code from the filesystem and/or pulling a Change Request from Azure DevOps, then`

`writing structured notes to the Obsidian vault. Output serves both technical`

`developers and non-technical stakeholders.`

`---`

`## Tools Required`

`| Tool | Purpose |`

`|------|---------|`

``| `view` / `bash_tool` | Read codebase from filesystem |``

``| `azure-devops` MCP | Pull CR / work item details from ADO |``

``| `mcp-obsidian` | Write documentation to Obsidian vault |``

`---`

`## Step 1 — Gather Inputs`

`Before reading any code, clarify these with the user if not already stated:`

`### 1a. Codebase Location`

`Ask for the **folder path** on the filesystem where the relevant code lives.`

`If the user gives a screen name or aggregate name (e.g., "Owner Statement"), use`

`` `bash_tool` to search for it: ``

` ```bash `

`# Find Angular component`

`find /path/to/repo -type f -name "*.ts" | xargs grep -l "OwnerStatement" 2>/dev/null`

`# Find C# aggregate or service`

`find /path/to/repo -type f -name "*.cs" | xargs grep -l "OwnerStatement" 2>/dev/null`

`# Find SQL objects`

`find /path/to/repo -type f \( -name "*.sql" -o -name "*.cs" \) | xargs grep -l "OwnerStatement" 2>/dev/null`

` ``` `

``Read the discovered files with `view`. Prioritize:``

``- **Angular**: component `.ts`, template `.html`, service `.ts`, any validators or guards``

`- **C# API**: aggregate root, domain services, application commands/queries (CQRS), validators, repository interfaces`

`- **SQL**: stored procedures, views, table definitions referenced by the feature`

`- **WinForms**: form code-behind, business logic classes, any sync/offline logic`

`### 1b. ADO Work Item (for CRs)`

`If a CR / work item is involved, ask for the **ADO work item ID or URL**.`

``Then use the `azure-devops` MCP tool to fetch it:``

` ``` `

`azure-devops: wit_get_work_item → id: <work_item_id>`

` ``` `

`Extract from the ADO item:`

`- **Title** — the CR name`

`- **Description** — what the customer wants`

`- **Acceptance Criteria** — the definition of done`

`- **Repro steps / comments** — any clarifications`

`- **Related work items** — parent feature, linked bugs, child tasks`

`- **Attachments or links** — mockups, references`

`Also fetch child tasks if any:`

` ``` `

`azure-devops: wit_get_work_items_batch_by_ids`

` ``` `

`### 1c. Scope Clarification`

`Confirm with the user:`

`- Which **Application** this belongs to (e.g., Construction API, Angular ERP, Ready Mix Desktop)`

`- Which **Module or Cycle** (e.g., Owner Statement, Delivery Tracking, Payroll Cycle)`

``- Which **Aggregate** if known (e.g., `OwnerStatementAggregate`, `ContractAggregate`)``

`- Is this for **existing documentation** (read code → document) or **CR documentation** (read code + ADO → spec)?`

`---`

`## Step 2 — Analyze the Code`

`Read systematically. For each layer found, extract:`

`### Business Rules (most important)`

`Look for:`

``- `if` conditions that gate business operations — document each as a rule``

`- Validation classes (FluentValidation, Angular validators, SQL CHECK constraints)`

`- Guard clauses at the start of methods`

``- Exceptions thrown with business messages (e.g., `throw new DomainException("Cannot post a closed period")`)``

`- Status/state machine transitions`

`- Role/permission checks`

`Write each rule in plain language:`

`` > ❌ `if (contract.Status != ContractStatus.Active) throw ...` ``

`> ✅ "A retention cannot be released unless the contract is in Active status."`

`### Data Flow`

`Trace the full path for the main operation(s):`

`- Angular form → API call → Command/Query → Aggregate → Repository → DB → Response → UI update`

`- Note which data is computed vs. persisted`

`- Note any caching, flags, or async operations`

`### Entity Relationships`

`From the aggregate root and DB schema, identify:`

`- Core entities and their relationships (one-to-many, etc.)`

`- Key foreign keys and their business meaning`

`- Any denormalized/cached fields and why they exist`

`### CR Impact Analysis (if ADO item is present)`

`After reading the code and the ADO item together, identify:`

`- Which files/classes will need to change`

`- Which business rules may conflict with or be extended by the CR`

`- Risk areas (shared aggregates, cross-module dependencies, permission impact)`

`- Questions or gaps in the CR description that need clarification before implementation`

`---`

`## Step 3 — Write Documentation to Obsidian`

``Use `mcp-obsidian` to write all notes. Follow this vault structure strictly:``

` ``` `

`Acclfex Docs/`

`└── {Application}/`

`└── {Module or Cycle}/`

`├── _overview.md ← Module-level, non-technical first`

`└── {Aggregate}/`

`├── _aggregate-overview.md ← Aggregate summary + all rules`

`└── {MethodOrFeature}.md ← Deep-dive per operation or CR`

` ``` `

`**Examples:**`

` ``` `

`Acclfex Docs/Construction API/Owner Statement/`

`_overview.md`

`OwnerStatementAggregate/`

`_aggregate-overview.md`

`CalculateRetention.md`

`ReleaseRetention.md`

`Acclfex Docs/Angular ERP/Owner Statement/`

`_overview.md`

`OwnerStatementComponent/`

`_aggregate-overview.md`

`OwnerStatementScreen.md`

` ``` `

`---`

`## Step 4 — Document Templates`

`Use these templates for each note type. Adapt sections based on what's available.`

``### `_overview.md` — Module Overview (Non-Technical First)``

` ```markdown `

`# {Module Name} — Overview`

`> **Application:** {App name}`

`> **Last Updated:** {date}`

`> **Related ADO:** {link if CR}`

`## What is this?`

`{2–3 sentences explaining what this module does in business terms, no jargon.`

`Who uses it? What problem does it solve?}`

`## Key Concepts`

`- **{Term}**: {plain-language definition}`

`- **{Term}**: {plain-language definition}`

`## Business Flow Summary`

`{Step-by-step in plain language — what happens from the user's perspective}`

`1. The project manager opens the Owner Statement screen…`

`2. The system calculates the retention amount based on…`

`3. Finance approves and the statement is posted…`

`## Modules & Aggregates Inside`

`| Aggregate / Component | Responsibility |`

`|----------------------|---------------|`

``| `OwnerStatementAggregate` | Holds all retention and billing logic |``

``| `OwnerStatementComponent` | Angular screen, handles display and user input |``

`## Known Business Constraints`

`- {Constraint 1}`

`- {Constraint 2}`

`## Open Questions / Gaps`

`- {Any unclear business rules found in code}`

` ``` `

`---`

``### `_aggregate-overview.md` — Aggregate / Component Deep Dive``

` ```markdown `

`# {AggregateName} — Technical Overview`

`> **Layer:** {API Aggregate / Angular Component / SQL / WinForms}`

`` > **Namespace / Path:** `{full class path or file path}` ``

`> **Last Updated:** {date}`

`## Responsibility`

`{What this aggregate/component owns. What it must NOT do (boundary).}`

`## Domain Entities`

`| Entity | Role | Key Fields |`

`|--------|------|-----------|`

``| `OwnerStatement` | Root | Id, ContractId, Status, TotalAmount |``

``| `RetentionLine` | Child | Percentage, ReleasedAmount, ReleaseDate |``

`## Business Rules`

`### Rule 1: {Rule Name}`

`` - **Source:** `{ClassName}.cs : line ~{N}` or `{component}.ts` ``

`- **Plain language:** {What it means in business terms}`

`` - **Code logic:** `{condensed condition}` ``

`- **Violation result:** {What happens if broken — exception, UI error, etc.}`

`### Rule 2: …`

`## State Machine (if applicable)`

` ``` `

`Draft → Submitted → Approved → Posted → Closed`

`↑ only Finance role can approve`

`↑ posting locks all lines`

` ``` `

`## Key Methods / Operations`

`| Method | Trigger | Description |`

`|--------|---------|-------------|`

``| `CalculateRetention()` | On statement save | Computes retention % per contract terms |``

``| `ReleaseRetention()` | Finance approval | Marks retention lines as released |``

`## Dependencies`

`- **API calls / Repos:** {what it reads or writes}`

`- **Other aggregates:** {any cross-aggregate calls — flag as design risk if present}`

`- **External:** {any third-party or integration points}`

` ``` `

`---`

``### `{MethodOrFeature}.md` — Operation or CR Spec``

` ```markdown `

`# {Method Name or CR Title}`

`> **Type:** {Existing Feature Doc / CR Spec}`

`> **ADO Work Item:** [{id}]({url})`

`> **Application:** {app}`

`` > **Aggregate:** `{AggregateName}` ``

`> **Last Updated:** {date}`

`## Summary`

`{One paragraph: what this does or what the customer wants to change.}`

`## Business Context`

`{Why does this exist? What business problem does it solve or what is being requested?}`

`## Acceptance Criteria`

`{From ADO if CR — paste in plain language, not raw ADO markup}`

`- [ ] {Criterion 1}`

`- [ ] {Criterion 2}`

`## Current Behavior (for CRs)`

`{Describe what the system does today, before the change.}`

`## Required Change (for CRs)`

`{Describe exactly what needs to change, informed by reading the code.}`

`## Business Rules Involved`

`| Rule | Status | Notes |`

`|------|--------|-------|`

`| Retention cannot be released on Closed contracts | ✅ Keep | No change needed |`

`| Retention % must match contract terms | ⚠️ Changing | CR extends this for partial release |`

`## Technical Implementation Plan`

`### Files to Change`

`| File | Change Type | Description |`

`|------|------------|-------------|`

``| `OwnerStatementAggregate.cs` | Modify | Add partial release logic |``

``| `ReleaseRetentionCommandHandler.cs` | Modify | Handle new partial flag |``

``| `owner-statement.component.ts` | Modify | Show partial release UI |``

``| `V20250520_AddPartialRetention.sql` | New | Migration for new column |``

`### Step-by-Step`

`1. {Implementation step 1}`

`2. {Implementation step 2}`

`## Risk & Impact`

`- {Risk 1 — e.g., shared aggregate used by another module}`

`- {Risk 2 — e.g., affects existing reports}`

`## Open Questions`

`- [ ] {Question 1 for PO or stakeholder}`

`- [ ] {Question 2 for tech lead}`

`## Test Scenarios`

`| # | Scenario | Expected Result |`

`|---|---------|----------------|`

`| 1 | Release retention on Active contract | Should succeed |`

`| 2 | Release retention on Closed contract | Should fail with message |`

` ``` `

`---`

`## Step 5 — Quality Checklist`

`Before finishing, verify:`

`- [ ] Every business rule from the code is written in plain language (no code jargon in the rule description)`

``- [ ] State machines are visualized if the entity has a `Status` field``

`- [ ] All CR acceptance criteria are mapped to specific code changes`

`- [ ] Open questions are explicitly listed — don't assume missing business rules`

`- [ ] Cross-module dependencies are flagged as risks`

`- [ ] Notes are saved to the correct Obsidian path hierarchy`

``- [ ] `_overview.md` is readable by a non-technical project manager``

`---`

`## Important Patterns to Watch For (Acclfex-Specific)`

`### Angular (accflex ERP frontend)`

`- Business rules are often duplicated in both the Angular component and the API — **document both and flag discrepancies**`

``- Look for `PermissionGuard`, `canActivate`, and `*ngIf` based on roles for authorization rules``

``- Watch for `FormGroup` validators — these are business rules too``

`### C# API (DDD / CQRS pattern)`

``- Aggregates own business rules — `Domain/` folder is the most important``

``- `Application/` layer (Commands, Queries, Handlers) shows the use-case flow``

`- FluentValidation classes are a goldmine of business rules — read every validator`

`- Any method on an aggregate that's not a simple getter is likely a business operation worth documenting`

`### SQL`

`- Stored procedures often contain legacy business logic — read them carefully`

`- Views may encode reporting rules`

`- CHECK constraints and triggers are business rules enforced at DB level`

`### WinForms (offline desktop apps)`

`- Business logic may be mixed with UI code — extract and document it separately`

`- Sync/offline rules are critical — document what happens when offline vs. connected`