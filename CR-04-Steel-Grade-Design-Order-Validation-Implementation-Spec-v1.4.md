# Implementation Specification — CR-04 Steel Grade Verification Against Design Order

| Field | Value |
|---|---|
| Spec ID | IS-CR-04 |
| Spec version | **1.4** — correction release aligned to platform 0.22.2 (supersedes 1.3) |
| Requirement | CR-04 — Steel grade verification against DO (Design Order) |
| Business feature name | Steel Grade Design-Order Validation |
| Feature kind | `Validation` (read-only) — `CimatronCustomization.DesignChecks` |
| Platform baseline | `cimatron-customization` **0.22.2** (Phase 02, Step 08B CLOSED / USER VALIDATED). Production assembly metadata `0.22.0.0` (see C-01). |
| Authoritative requirement source | *Customer Requirement Document — Mold Design Checklist Automation in Cimatron* (Sheet 1), row CR-04 + §3 cross-cutting requirements |
| Supporting sources | Day-1 meeting transcript 05-Aug-2026 (00:12:32–00:16:06); `docs/platform/STEP-06-*`; `docs/platform/CIMATRON-2026-API-CAPABILITY-VERIFICATION.md` (U7, U8, U12, U13, U14, U15, U17); Step 08B closure evidence (0.22.0–0.22.2); official Cimatron API documentation via the `cimatron-api` MCP server |
| Spec status | **DRAFT — input to the implementing agent.** Not an implementation approval. The agent must follow `AGENTS.md` §33 (study → plan → file list → wait for explicit approval) for every slice. |
| Open decisions | 17 open decisions (D-01…D-18, with D-13 resolved). **BLOCKING** items must be answered before the corresponding slice is approved. |

### Changes from 1.3

| Area | Change |
| Evidence boundary | `GetAttribute(cmAttString, name)` is verified only for **present string attributes**. Missing-name and same-name/non-string behaviour are now explicit Slice 2A capability checks; v1.2 no longer claims `Absent`/`UnsupportedType` runtime semantics without evidence. |
| Placement identity / D-15 | Tightened highlight safety to the production highlighter's actual search behavior: a normal reference is emitted only when its `InstanceId` is unique across the full unfiltered highlighter-equivalent traversal. Suppressed/excluded occurrences still participate in this safety check. Unsafe references are omitted while business findings keep their own status. |
| DO blank grade | `SteelGrade` is transport-optional so a key+blank-grade row reaches Domain. New D-16 decides whether that is an Error or a deliberate exclusion; the generic tabular `TABULAR_REQUIRED_VALUE_MISSING` is not used to decide this business meaning. |
| Option B/C | BOM material is evidence only; **assembly traversal is still required** for placement/suppression/highlight context. `cmNoFiltering` suppressed-occurrence behaviour and blank `Standard Number` handling are gated by live evidence / D-18. |
| Path resolution | Existing `CimatronSessionSnapshot` has no assembly title/path. D-09(b) now requires an explicitly verified assembly-identity source/port; no token source is assumed. |
| Findings/logging | `Finding` has no Code property, so stable CR-04 codes are message prefixes unless a separately approved platform change is made. The existing executor logs one `Findings=<count>` plus report message; feature-internal DO/occurrence counts are not claimed as logger fields. |
| Live validation | Adds top-level highlight, suppressed-parent nested occurrence, missing-attribute, non-string attribute, assist-part policy, and repeated-placement resolution checks. |
| Corrections | Corrects `SubInstances` documentation wording, CSV 08B scope, and ST06-DF-05 ownership/meaning. |
| Slice 2A governance | Defines 2A as an in-process Plugin diagnostic with separate KA-model preparation approval, its own governed 0.23.x package/evidence, and ordering before or alongside Slice 2. |
| Highlight status semantics | Clarifies that unsafe highlight identity never changes a valid business finding from `Passed`/`Failed`/`Warning`; the reference is omitted instead. |

---

## 1. Requirement

### 1.1 Customer requirement (authoritative, from the CRD)

> The steel grade for each block (e.g. "1.2738") is defined in the customer's Excel Input Sheet and engraved on the blocks. The software must compare the steel grade in the CAD model with the Excel Input Sheet and flag mismatches.
>
> **Agreed mechanism:** the customer adds a material / manufacturing attribute on each component; Cimatron reads the attribute and compares it with the Excel entry, reporting any mismatch. **Disposition: To develop (API) – attribute-based.**

Cross-cutting requirements that apply (CRD §3): Excel Input Sheet is master data ("Excel to CAD" verification); attributes are the identification mechanism; one-click execution with NG highlighting; rule values editable, not hard-coded.

### 1.2 Interpretation used by this spec

For every mold component listed in the Design-Order (DO) Input Sheet, the feature reads the expected steel grade from the sheet, reads the steel grade carried by the corresponding CAD component, compares them under an approved comparison policy, and reports `Passed` / `Failed` per component, with the CAD component highlightable from the Result Report. The overall feature result is the canonical aggregation of all findings.

### 1.3 Transcript context (supporting only — CRD wins on conflict)

- Engraving on blocks reads e.g. `1.2738 PCK` — grade token `1.2738`; suffix is a manufacturing designation (see D-06).
- The customer confirmed the design is moving to Cimatron and that attributes can be added — attribute-based identification is the customer-accepted route.
- Day-2 confirmed engravings are real manufactured engravings. Verifying the **engraved text** is **out of scope** (§2.2).

---

## 2. Scope

### 2.1 In scope

1. Read the DO Input Sheet (`.xlsx` / `.xls` / `.csv`) through the existing Step 06 pipeline (`ReadMappedTabularData` + `ExcelTabularDataReader`).
2. Traverse the active **Assembly** (including nested sub-assemblies) and obtain each component's steel grade through the route chosen in D-01.
3. Match DO rows to CAD components by the key chosen in D-04.
4. Normalize and compare grades under the approved policy (D-05, D-06, D-07).
5. Produce per-component findings with highlightable entity references and an aggregated outcome.
6. Resolve the Step 06 deferred findings this CR owns (§15) and the 08B carry-forward items assigned to CR-04 (§4).

### 2.2 Out of scope

| Item | Reason |
|---|---|
| Verifying the **engraved text** (`1.2738 PCK`) | Engraved-text read-back is not verified. Future extension. |
| Writing/creating attributes on components | CR-04 is read-only. Attribute authoring is customer practice (CRD action #8). |
| Reading an **instance/component-level** attribute | U14 — `IAssInstance.Attribute` is SET-ONLY (installed Interop and official docs agree). Not available. |
| Reading the Part material property directly | U17 — no evidence-backed getter. Only reachable through the BOM (U15). |
| `cmVisibleOnly` BOM semantics | 08B verified `cmNoFiltering` only. CR-04 must not use `cmVisibleOnly`. |
| Part-document context | Assembly-only (D-12). |
| Parameter-editing UI; Zoom-to-result | Unchanged platform limitations. |

---

## 3. Platform baseline (0.22.2)

CR-04 is a consumer of existing, validated contracts. No new framework.

| Platform element | Status in 0.22.2 | Use in CR-04 |
|---|---|---|
| `IApplicationFeature`, `FeatureDefinition`, `FeatureTraceability`, `FeatureRegistry`, `FeatureComposition` | Validated | Registration and metadata |
| `ExecuteRegisteredFeature` pipeline | Validated | Execution, context check, parameter resolution, logging |
| Schema-2 rule parameters | Validated | All customer values (§9) |
| `ReadMappedTabularData` + `ExcelTabularDataReader` | `ReadMappedTabularData` validated in-process (06E); 08B additionally exercised `ITabularDataReader.Read` directly on CSV (not the full mapped pipeline) | DO sheet read; BOM CSV read (Option B/C) |
| `Finding`, `FindingAggregation`, `ResultStatus` | Validated | Result model |
| `IAssemblyStructureReader` | Validated | Hierarchy, suppression |
| **`IViewportEntityHighlighter` / `CimatronViewportEntityHighlighter`** | **Single-placement nested assembly reference supported and IN-PROCESS VERIFIED (U12)** via the production Result Report Highlight | NG highlight only when the CR-04 reference is proven unambiguous |
| **`IAssemblyEntityMapper.MapToActiveAssembly(ModelEntityReference)`** | IN-PROCESS VERIFIED (U7) | Reusable as-is if a mapped reference is needed |
| **`IOccurrenceEntityAttributeReader.ReadNestedOccurrenceAttribute(expectedAssemblyTitle, targetModelTitle, bodyId, attributeName)`** | IN-PROCESS VERIFIED (U8/U13) — **known-answer-shaped** | **Not reused directly** (one model, one known body ID, nested-only). Its verified routines are the template for the CR-04 general reader (§8). |
| **`IBomMaterialExporter.ExportAndReadMaterial(expectedAssemblyTitle, targetModelTitle)`** | IN-PROCESS VERIFIED (U15, `cmNoFiltering`) — **known-answer-shaped** | **Not reused directly** (one target row). Template for the CR-04 BOM table reader (§8, Option B/C only). |
| `IEntityAttributeReader` (enumeration) | Validated for enumeration only | **Not used** — relies on returned `IAttribute.Name`, which Cimatron returns blank (07C-3, reconfirmed 08B). |

### 3.1 Capability status

| ID | Capability | Status | CR-04 use |
|---|---|---|---|
| U8 | Occurrence traversal incl. nested (`GetInstances` / `GetInstancesByModel` / `SubInstances`) | **IN-PROCESS VERIFIED** (3 total, 2 top-level, 1 nested on KA-ASM-1) | Option A, highlight |
| U13 | `IAssInstance.GetAllEntities()` + `GetAttribute(cmAttString, name)` | **IN-PROCESS VERIFIED**; `GetAllEntities()` **undocumented in official SDK** (installed Interop + runtime evidence only) | Option A |
| U7 | `IAssemblyModel.GetAssemblyEntity` | **IN-PROCESS VERIFIED** | Highlight path |
| U12 | Nested temporary highlight via production Result Report | **IN-PROCESS VERIFIED** | NG highlight |
| U15 | `IBOM.Export` `cmNoFiltering` + `Material` read through `ITabularDataReader` | **IN-PROCESS VERIFIED for `cmNoFiltering` material read** | Option B / C |
| U14 | Instance attribute getter | NOT AVAILABLE | Never |
| U17 | Part material getter | NOT AVAILABLE | Never |

Verification was on `KA-ASM-1` (one nested level, one placement of the sub-assembly). Behaviour with **missing named attributes, same-name attributes of a different type, repeated sub-assembly placements, placement-specific mapping, deeper nesting, assist parts and large assemblies is not yet evidenced** — covered by D-15–D-18, R-11–R-16 and the §13.5 known-answer model.

---

## 4. Carry-forward items from the Step 08B review

| ID | Item | Resolution in CR-04 |
|---|---|---|
| C-01 | **Version metadata drift.** 0.22.1 changed `CimatronBomMaterialExporter` but production `AssemblyInfo` stayed `0.22.0.0`, so two different CimatronIntegration binaries share one version. | The first CR-04 slice that changes production code bumps metadata of **every** changed production assembly; Slice 3 (CimatronIntegration change) must bump CimatronIntegration. `VERSION.md` records that 0.22.1 binaries were mis-versioned. |
| C-02 | **Undocumented second live execution.** 08B evidence shows two correlations (`ecc1f44b…` Result Report, `508db3e5…` Highlight screenshot) while closure documents one governed run. | Before Slice 1 approval, the agent appends a note to the 08B handoff/history recording both correlations and why. For CR-04 live runs: every execution's correlation ID is recorded in the evidence, including repeats. |
| C-03 | **"Passed before confirmation" pattern.** The 08B U12 row was `Passed` when the reference was only *ready*. | **Prohibited in CR-04.** A finding is `Passed` only when its check has been performed. Highlight availability is never a finding. |

---

## 5. Open decisions

Each decision has a **recommended default**. A recommendation is not an approval.

| ID | Decision | Options | Recommendation | Owner | Blocking for |
|---|---|---|---|---|---|
| **D-01** | CAD-side evidence source | **A** body-level named string attribute (U13 present-string route). **B** BOM `Material` column (U15). **C** Both, cross-checked. | **A** — matches the CRD-agreed mechanism and avoids BOM temp-file/GetBomManager exposure. **B/C still require assembly traversal** for placement, suppression and highlight context; U15 verifies material read, not complete CR-04 semantics. | Customer + user | Slice 2, 3 |
| **D-02** | Attribute name and runtime type | Customer-defined name; intended type `cmAttString` | Name = config parameter (no default). Present-string lookup is verified. Missing-name and same-name/non-string runtime behaviour must be observed in the Slice 2A capability investigation before `Absent` / `UnsupportedType` mapping is finalized. | Customer + user | Slice 3 |
| **D-03** | DO Input Sheet layout | Worksheet, header row, key header, grade header | All config parameters; no invented defaults. Real customer sample required before acceptance. | Customer | Slice 4, acceptance |
| **D-04** | Row ↔ component key | **(a)** component `ModelTitle`. **(b)** second configured body string attribute. **(c)** BOM `Standard Number` (B/C only). | **(a)** unless the real DO uses item/catalogue numbers. Matching = trimmed, whitespace-collapsed, case-insensitive ordinal. | Customer | Slice 1, 2 |
| **D-05** | Grade representation (**ST06-DF-03 mandatory**) | **Strict** text policy; **WerkstoffCanonical**; optional future reader-level numeric-cell guard (protected-file change). | **Strict** + text-formatted grade column first. Regex ambiguity guard remains the no-platform-change option; raw-cell-type guard may be proposed separately if customer data requires it. | Customer + user | Slice 1 |
| **D-06** | Suffix handling (`1.2738 PCK`) | Full string; grade token only | **Full string**; token mode as Boolean parameter. | Customer | Slice 1 |
| **D-07** | Normalization | Trim; collapse whitespace (incl. NBSP, tab); case-insensitive | All three, both sides; nothing else stripped. | User | Slice 1 |
| **D-08** | Coverage rules | §7.3 | DO row without component → `Failed`; attributed/material-bearing component not in DO → `Warning`; neither → not reported. Deliberate exclusions are governed by D-16/customer rules, not guessed. | Customer + user | Slice 1 |
| **D-09** | DO file path supply | (a) absolute-path parameter; (b) `{AssemblyFolder}` / `{AssemblyTitle}` pattern; (c) launcher file dialog | No source for assembly folder/title exists in `CimatronSessionSnapshot`. **(a)** is immediately implementable and suitable for development / known-answer runs, but editing `config.xml` for every mold is an operational cost and should not become the production default without customer approval. **(b)** requires a new read-only assembly-identity port plus verified Cimatron API route before approval. | Customer + user | Slice 2, 4 |
| **D-10** | Tabular error reporting (**ST06-DF-01**) | First-error; multi-error | First-error, `ACCEPTED for CR-04`; CR-22 re-evaluates. | User | Slice 2 |
| **D-11** | Header whitespace (**ST06-DF-04**) | Strict; trim | Decide from real workbook; trimming is a protected-file change to `TabularMappingEngine`/reader. | User | Slice 2 |
| **D-12** | Document context | Assembly only; Part also | **Assembly only.** | User | Slice 4 |
| ~~D-13~~ | ~~Nested highlight fallback~~ | — | **RESOLVED only for the single-placement nested case verified by 08B.** Placement ambiguity remains D-15. | — | — |
| **D-14** | Multi-body components; duplicate DO keys | §7.2 | 0 values → missing; 1 → value; >1 distinct → `Failed` ambiguous. Duplicate DO key → `Error`. | Customer + user | Slice 1 |
| **D-15** | **Occurrence identity and highlight safety.** The production highlighter searches instances by `InstanceId` in depth-first order, stops at the first match, and only then checks the target model PID. Therefore safety depends on `InstanceId` uniqueness across the **full unfiltered occurrence traversal equivalent to the highlighter's search scope**, not only CR-04 candidates and not only `(InstanceId, ModelPid)` uniqueness. Suppressed, excluded and other non-CR-04 occurrences still participate in this safety check. | **(a)** Emit a normal production highlight reference only when its `InstanceId` is unique across the full highlighter-equivalent traversal; otherwise keep the business finding and omit the unsafe reference, with a message that highlight is unavailable because occurrence identity is ambiguous. **(b)** If a documented/runtime-verified placement-specific route exists, add it under a separately approved platform increment. **(c)** Accept first-match highlight is **not allowed**. | **Run a §31 API investigation in Slice 2A** on `KA-ASM-CR04`, including repeated placement, an active same/local-ID collision and a suppressed-collision control; test documented placement candidates and measure collision frequency on a customer-like assembly. Choose (a) unless (b) is proven; if collisions are common enough to undermine the CRD NG-highlight requirement, escalate (b) as functionally required. | User | Slice 2A gate |
| **D-16** | DO row with component key but blank steel grade | Error; deliberate exclusion/not-applicable; other customer-defined meaning | **Do not let the tabular required-field rule decide this.** Read grade as optional text, then apply the approved Domain policy. If blank is invalid, emit stable `DO_GRADE_MISSING` Error. | Customer + user | Slice 1, 2 |
| **D-17** | Assist parts | Include; exclude | **Exclude** unless customer says assist parts belong in the DO. Verify how top-level enumeration exposes assist parts; `get_SubInstances(1,0)` explicitly excludes them. | Customer + user | Slice 2A gate |
| **D-18** | BOM row with blank `Standard Number` (B/C only) | Ignore with evidence; Warning; Error | No default until a real BOM sample / live run shows whether such rows occur. `Standard Number` must not be declared required at reader level until this is decided. | User | Slice 2/3 if B/C |

---

## 6. Functional behaviour

### 6.1 Run flow

```text
1. Pipeline pre-checks (existing): Cimatron available; active document = Assembly (else NotApplicable);
   parameters resolved/validated (missing required -> Error, no defaults).
2. Resolve DO file location:
   - D-09(a): use configured absolute path.
   - D-09(b): first read assembly identity through the separately approved/verified assembly-identity port,
     then resolve {AssemblyFolder}/{AssemblyTitle}. No source is inferred from CimatronSessionSnapshot.
   Missing file -> Error TABULAR_FILE_NOT_FOUND.
3. Read DO sheet via ReadMappedTabularData: ComponentKey (Text, required), SteelGrade (Text, optional at transport level).
   Any tabular discovery/mapping/read failure -> Error with stage + stable code; stop.
4. Validate DO rows (Domain): duplicate keys -> Error; D-05 ambiguity -> Error; D-16 blank-grade policy;
   zero rows -> Error; stop on Error.
5. Traverse assembly context for every CR-04 route (A/B/C): placement path, model title, suppression/inherited
   suppression, body/entity references, and assist-part policy. Any API/runtime failure -> Error; stop.
6. Read CAD grade evidence:
   Option A: named body attribute observations. Missing-name / non-string behaviour follows the verified
             Slice 2A capability result; it is not assumed.
   Option B: cmNoFiltering BOM table -> {Standard Number, Material}; merge BOM evidence onto assembly
             traversal by the approved key. Blank Material is data; blank Standard Number follows D-18.
   Option C: both sources, then cross-check.
7. Group by normalized component key; carry only references proven safe for the specific placement.
   Ambiguous repeated placements follow D-15 and must never silently first-match.
8. Evaluate (Domain, pure) -> findings (§7.3).
9. Aggregate with FindingAggregation.
10. Result Report lists findings; Highlight uses only safe current-session references.
```

### 6.2 Invariants

- **Read-only.** Option A writes nothing. Option B/C writes only the governed temp CSV (§11).
- **Business result is independent of highlightability.** COM/runtime failures and unreadable inputs are `Error`. A valid steel-grade comparison remains `Passed` / `Failed` / `Warning` according to §7.3 even when no safe entity reference can be emitted. Unsafe/unproven highlight targeting therefore removes the reference and adds an explanatory message; it does **not** rewrite the business finding to `Unsupported`.
- **No invented defaults.** Every customer value comes from configuration / approved decision (§9).
- **Suppression:** suppressed occurrences and descendants of suppressed sub-assemblies are skipped. Hidden-but-not-suppressed occurrences are evaluated. For B/C, `cmNoFiltering` suppression semantics must be observed before acceptance.
- **Assist parts:** governed by D-17; current nested call `(includeSuppressed=1, includeAssist=0)` excludes assist parts.
- **No persisted entity references.** References are current-session only.
- **D-15:** a normal production highlight reference is emitted only when its `InstanceId` is unique across the full unfiltered occurrence traversal equivalent to the production highlighter's search scope, unless a separately verified placement-specific resolver supersedes that rule. Suppressed/excluded occurrences still participate. No ambiguous reference may silently target an arbitrary occurrence.
- **C-03:** no finding is `Passed` unless its comparison was actually made.

---

## 7. Domain model and rules

### 7.1 Pure types (Domain — no Cimatron, no IO)

| Type | Responsibility |
|---|---|
| `SteelGradeRepresentation` | Enum `Strict`, `WerkstoffCanonical` |
| `SteelGradeComparisonPolicy` | Representation + `CompareGradeTokenOnly` |
| `SteelGradeNormalizer` | §7.2 rules → `NormalizedSteelGrade` or ambiguity result |
| `NormalizedSteelGrade` | Original text + comparison key; equality ordinal-ignore-case |
| `ComponentKeyNormalizer` | Trim + collapse whitespace + case-insensitive key |
| `DesignOrderSteelGradeEntry` | Key, original key text, grade (may be absent until D-16 policy is applied), source row number |
| `CadComponentSteelGradeEvidence` | Key, original key, distinct grade values, body count, placement descriptions, `IReadOnlyList<ModelEntityReference>` |
| `SteelGradeDesignOrderEvaluation` | Pure `Evaluate(entries, evidence, policy)` → `IReadOnlyList<Finding>` |

### 7.2 Normalization and comparison rules

1. `null`/whitespace → *absent* in the pure Domain model; absent never equals anything (absent vs absent is not a match). This does **not** assert how Cimatron `GetAttribute` reports a missing named attribute; that is a Slice 2A runtime question.
2. Trim; replace any whitespace run (space, tab, U+00A0) with one space.
3. `CompareGradeTokenOnly`: keep text before the first space.
4. **Strict:** key = step-3 text. **WerkstoffCanonical:** if `^\d\.\d{1,4}$`, right-pad decimals with `0` to 4 digits.
5. **DO-side guard (Strict only):** DO value matching `^\d\.\d{1,3}$` → `DO_GRADE_AMBIGUOUS_NUMERIC` (Error). Reason: numeric Excel cells lose trailing zeros through invariant text conversion (ST06-DF-03).
6. Equality ordinal-ignore-case; no fuzzy or substring matching.
7. Multi-body (D-14): collect from every `cmBody`; distinct non-absent values 0 → missing, 1 → value, >1 → ambiguous.
8. **Option B/C:** a BOM row with blank `Material` yields *absent* material for that row; it is never an API/read error by itself (lesson from 08B 0.22.0 → 0.22.1). A blank `Standard Number` follows D-18.

Grades are compared strictly as text; no floating-point comparison exists (`AGENTS.md` §25 satisfied).

### 7.3 Finding catalogue

`RequirementId = "CR-04"`; messages show original (non-normalized) text. `Finding` in 0.22.2 has **no Code property**, so each stable catalogue code is emitted as a message prefix such as `[STEEL_GRADE_MISMATCH] ...`. Changing `Finding.cs` is outside CR-04 unless separately approved.

| Code | Condition | Status | Entity refs |
|---|---|---|---|
| `STEEL_GRADE_MATCH` | DO row matched; single CAD value equals DO grade | `Passed` | Safe references only; ambiguous occurrence identity follows D-15 |
| `STEEL_GRADE_MISMATCH` | Single CAD value differs | `Failed` | Safe references only; ambiguous occurrence identity follows D-15 |
| `STEEL_GRADE_ATTRIBUTE_MISSING` | Matched component; no body carries the value after runtime semantics are verified (Option B: blank Material) | `Failed` | Safe references only |
| `STEEL_GRADE_ATTRIBUTE_AMBIGUOUS` | >1 distinct value across bodies | `Failed` | Safe references only |
| `DO_COMPONENT_NOT_FOUND` | DO key matches no non-suppressed occurrence | `Failed` | None |
| `CAD_COMPONENT_NOT_IN_DO` | Component carries a value but its key is not in the DO | `Warning` | Safe references only |
| `STEEL_GRADE_ATTRIBUTE_TYPE_UNSUPPORTED` | **Only if Slice 2A proves this state is observable** for the requested name | `Unsupported` | Safe placements only |
| `DO_DUPLICATE_COMPONENT_KEY` | Two DO rows normalize to one key | `Error` (stop) | None |
| `DO_GRADE_AMBIGUOUS_NUMERIC` | §7.2 rule 5 | `Error` (stop) | None |
| `DO_GRADE_MISSING` | Key present, grade blank, and D-16 = invalid | `Error` (stop) | None |
| `NO_DO_ROWS` | Zero data rows | `Error` (stop) | None |

Option C (both sources) adds `STEEL_GRADE_SOURCE_DISAGREEMENT` (`Failed`) when attribute and BOM values differ for one component.

Outcome = `FindingAggregation.Aggregate(findings, ResultStatus.Passed)`; outcome message gives counts per status only (§12).

---

## 8. Architecture and file plan

Placement follows `docs/platform/CR-PLATFORM-GUIDE.md`. The agent presents the final file list for approval per slice.

### 8.1 Port strategy

The 0.22.2 ports are capability-named but their **signatures are known-answer-shaped**. CR-04 must not change the verified 08B adapters. New general ports are introduced only after the required Slice 2A capability checks.

| Need | Approach |
|---|---|
| Assembly traversal / placement context (all A/B/C) | **New general port** `IAssemblyBodyAttributeReader` for A/C, or a traversal snapshot shared with B/C. It carries occurrence path, model title, suppression state and body/entity references. The repeated-placement identity model is finalized only after D-15 investigation. |
| Named grade/key attributes (A/C) | Use the same verified **present-string** route as 08B: `IAssInstance.GetAllEntities()` → `cmBody` → `IAttributeSink.GetAttribute(cmAttString, requestedName)`. Missing-name and wrong-type mappings remain provisional until smoke-tested. |
| BOM material rows (B/C only) | `IBomMaterialTableReader` + adapter following the verified `GetBomManager()` / `IBOM.Export(..., cmNoFiltering, ...)` route. `Material` is optional per row. `Standard Number` is also reader-optional until D-18 is decided. BOM rows are merged with traversal context; BOM alone does not provide highlight/suppression/placement identity. |
| Assembly identity for D-09(b) | New read-only `IActiveAssemblyIdentityReader` (name provisional) returning assembly title and document path/folder. **No Cimatron API route is assumed in this spec**; it must be verified under `AGENTS.md` §31 before D-09(b) implementation. Not needed for D-09(a). |
| Highlight | Existing `IViewportEntityHighlighter` stays unchanged for already-safe references. CR-04 emits normal references only when `InstanceId` is unique across the full unfiltered highlighter-equivalent traversal, unless Slice 2A proves a placement-specific mapping route. |
| Mutation evidence | Same `MutationSnapshot` approach; B/C additionally guards model-folder contents and temp cleanup. |

**Code sharing decision:** verified 08B adapters keep their private helpers. CR-04 may introduce internal traversal/reference helpers for **new** adapters after D-15 is resolved. Do not refactor 08B code as part of CR-04. This technical debt is independent of ST06-DF-05.

### 8.2 New port contracts (provisional until Slice 2A closes D-15 / attribute semantics)

```csharp
public interface IAssemblyBodyAttributeReader
{
    ModelReadResult<AssemblyBodyAttributeSnapshot> ReadNamedAttributes(
        IReadOnlyList<string> attributeNames);
}

// Only if D-09(b) is approved and its Cimatron API source is verified.
public interface IActiveAssemblyIdentityReader
{
    ModelReadResult<ActiveAssemblyIdentity> ReadActiveAssemblyIdentity();
}

// Option B/C only.
public interface IBomMaterialTableReader
{
    BomMaterialTableResult ReadMaterialTable();
}
```

`AssemblyBodyAttributeSnapshot` must carry enough data to express the **observed** placement model. `OccurrencePath` is useful for messages/detection, but v1.2 does **not** claim that an ID path is sufficient to re-resolve a placement for highlighting.

`BodyNamedAttributeObservation` has requested `AttributeName`, `BodyEntityId`, safe `ModelEntityReference` when available, and an observation status finalized from the smoke evidence. Returned `IAttribute.Name` never populates `AttributeName`.

`BomMaterialTableResult` rows contain `StandardNumber` (may be empty pending D-18) and `Material` (may be empty), plus temp/no-mutation evidence.

### 8.3 Adapter rules and capability gates

**Already verified routes:**

| Purpose | API route | Evidence boundary |
|---|---|---|
| Top-level/nested model traversal | `IAssemblyModel.GetInstances()` plus the installed indexed `IAssInstance.get_SubInstances(includeSuppressed, includeAssist)` and sub-assembly model access | U8 on one nested placement; repeated-placement identity and assist enumeration not yet verified |
| Bodies | `IAssInstance.GetAllEntities()` → keep `cmBody` | U13 present model; undocumented in official SDK |
| Named string attribute | `IAttributeSink.GetAttribute(cmAttString, name)` | U13 **present-string only** |
| Existing nested highlight | current `CimatronViewportEntityHighlighter` / `GetAssemblyEntity` | U12 on one nested placement only |
| Suppression flag | `IAssInstance.IsSuppressed` | Step 04; inherited/repeated live cases still tested in CR-04 |
| BOM | `IAssemblyModel.GetBomManager()`, `IBOM.Export(..., cmNoFiltering, ...)` | U15 material read; suppressed-row semantics not established |

**Mandatory Slice 2A capability investigation:**

1. `GetAttribute(cmAttString, <missing-name>)`: observe null / exception / other result.
2. Same requested name stored as a non-string type: observe whether distinguishable from absent.
3. Occurrence-identity safety (D-15): test repeated `SA-SLIDER-UNIT` placements, a deliberate active same/local-`InstanceId` collision, and a collision where the earlier conflicting occurrence is suppressed; determine whether nested `IAssInstance` objects are placement-specific, whether `InstanceId` is unique across the full highlighter-equivalent traversal, and whether a particular placement can be mapped/highlighted safely. Measure collision frequency on a customer-like assembly and record the percentage/count of otherwise valid CR-04 findings that would lose highlight under rule (a).
4. Top-level NG highlight (`CAVITY_INSERT`).
5. Nested occurrence under a suppressed parent.
6. Assist-part enumeration/identity (D-17).
7. If B/C: `cmNoFiltering` behaviour for suppressed occurrences and at least one blank `Standard Number` case if available.
8. If D-09(b): identify/document the supported API that returns active assembly title and saved file path/folder.

**Forbidden (architecture-test enforced):**

- `partModel.GetEntityById(...)` for occurrence bodies;
- equality on returned `IAttribute.Name`;
- `IAssInstance.Attribute` getter;
- any attribute setter, `Save`, or model-modifying call;
- `cmVisibleOnly` or any BOM filter other than `cmNoFiltering`;
- `Material` required for every BOM row;
- `Standard Number` required at reader level before D-18;
- COM calls off the Cimatron UI thread;
- first-match highlight for any reference whose `InstanceId` is non-unique across the full unfiltered highlighter-equivalent traversal.

Unexpected COM exceptions propagate to the pipeline → `Error`. Missing-attribute/wrong-type mapping is defined **after**, not before, the smoke evidence.

### 8.4 New files (expected)

| Layer | File(s) |
|---|---|
| Domain | `SteelGradeRepresentation.cs`, `SteelGradeComparisonPolicy.cs`, `SteelGradeNormalizer.cs`, `NormalizedSteelGrade.cs`, `ComponentKeyNormalizer.cs`, `DesignOrderSteelGradeEntry.cs`, `CadComponentSteelGradeEvidence.cs`, `SteelGradeDesignOrderEvaluation.cs` |
| Application | `IAssemblyBodyAttributeReader.cs`, assembly evidence DTOs, `VerifySteelGradeAgainstDesignOrder.cs`, `DesignOrderInputPathResolver.cs`; **only for D-09(b)** `IActiveAssemblyIdentityReader.cs` + `ActiveAssemblyIdentity.cs`; Option B/C: `IBomMaterialTableReader.cs`, `BomMaterialTableResult.cs`, `BomMaterialRow.cs` |
| CimatronIntegration | `CimatronAssemblyBodyAttributeReader.cs`, internal helpers finalized after D-15; **only for D-09(b)** assembly-identity adapter after API verification; Option B/C: `CimatronBomMaterialTableReader.cs` |
| DesignChecks | `SteelGradeDesignOrderValidation.cs` |
| Tests | Domain, Application, DesignChecks-feature, CimatronIntegration/architecture tests (§13) |
| Docs | `docs/requirements/CR-04-STEEL-GRADE-DESIGN-ORDER-VALIDATION.md` (§16) |
| Test data | `tests/TestData/CR04/` fixtures |

### 8.5 Existing files expected to change (each needs explicit approval)

| File | Change |
|---|---|
| `src/CimatronCustomization.Plugin/FeatureComposition.cs` | Register the feature with approved adapter(s) and tabular reader |
| `config/CimatronCustomization.config.xml` | Add production feature parameters only; **KA/test-only values must live in test fixtures/scripts/evidence, not client config** |
| `tests/CimatronCustomization.ArchitectureTests/ArchitectureBoundaryTests.cs` | Extend guards to new adapters |
| `*.csproj` of changed projects | New file wiring |
| `Properties/AssemblyInfo.cs` of every changed production assembly | Version bump (C-01) |
| `docs/platform/STEP-06-DEFERRED-FINDINGS.md` | DF-01/03/04/06 status; DF-05 only if independently affected |
| 08B handoff / `IMPLEMENTATION-HISTORY.md` | C-02 correlation note |
| `VERSION.md`, `CHANGELOG.md`, `IMPLEMENTATION-HISTORY.md`, `CHECKSUMS.sha256`, timesheet, session handoff | Standard package records |
| *Only if D-11 = trim or D-05 reader-level numeric guard approved:* protected tabular reader/mapping files | Separate approval |
| *Only if D-15 proves and user approves a platform increment:* highlighter/reference producers | Separate platform step |

Unchanged unless separately approved: `CimatronOccurrenceEntityAttributeReader.cs`, `CimatronBomMaterialExporter.cs`, `CimatronAssemblyEntityMapper.cs`, `CimatronViewportEntityHighlighter.cs`, `Step08BInProcessConfirmationFeature.cs`, and everything not listed.

---

## 9. Configuration

Feature id `STEEL-GRADE-DESIGN-ORDER-VALIDATION`. Examples are **placeholders**; no defaults are coded. Known-answer/test values belong in test fixtures or governed validation scripts/evidence, not production `config.xml`.

| Parameter | Type | Required | Example (illustrative) | Notes |
|---|---|---|---|---|
| `DesignOrderPath` | String | Yes if D-09(a) | `D:\Orders\M123_DO.xlsx` | Absolute path; immediately implementable without new Cimatron identity API. Suitable for development/KA runs, but requires config editing per mold unless an external deployment convention supplies it. |
| `DesignOrderPathPattern` | String | Yes only if D-09(b) | `{AssemblyFolder}\{AssemblyTitle}_DO.xlsx` | Requires approved `IActiveAssemblyIdentityReader` and verified API source; tokens only after that gate |
| `DesignOrderWorksheetName` | String | Yes for Excel | `Input Sheet` | Ignored for `.csv` |
| `DesignOrderHeaderRow` | Integer | Yes, min 1 | `1` | |
| `DesignOrderComponentKeyHeader` | String | Yes | `Component` | Exact header unless D-11 changes it |
| `DesignOrderSteelGradeHeader` | String | Yes | `Steel Grade` | Column must exist, but per-row value is transport-optional and governed by D-16 |
| `SteelGradeSource` | String | Yes | `BodyAttribute` | `BodyAttribute`, `BomMaterial`, `Both` |
| `SteelGradeAttributeName` | String | Required unless `SteelGradeSource=BomMaterial` | `MATERIAL` | Intended `cmAttString`; runtime missing/wrong-type semantics verified by Slice 2A |
| `ComponentKeySource` | String | Yes | `ModelTitle` | `ModelTitle`, `Attribute`, `BomStandardNumber`; latter only with B/C |
| `ComponentKeyAttributeName` | String | Required if `ComponentKeySource=Attribute` | `ITEM_NO` | |
| `SteelGradeRepresentation` | String | Yes | `Strict` | `Strict`, `WerkstoffCanonical` |
| `CompareGradeTokenOnly` | Boolean | Yes | `false` | D-06 |

Exactly one D-09 path strategy is active. Cross-field inconsistencies and invalid enumerated text → `Error` naming the parameter and allowed values.

---

## 10. Cimatron API usage

| API | R/W | Official docs (MCP) | Platform status | Notes |
|---|---|---|---|---|
| `IAssemblyModel.GetInstances()` | Read | Documented | U8 in-process | Traversal |
| `IAssInstance.SubInstances` | Read | Official docs show indexed include-suppressed/include-assist access; installed Interop exposes `get_SubInstances(int,int)` → `Object` | U8 in-process | 08B used `get_SubInstances(1, 0)` = include suppressed, exclude assist; D-17 governs CR-04 policy |
| `IAssInstance.Model`, `IsSuppressed`, `Id` | Read | Documented | Validated | |
| **`IAssInstance.GetAllEntities()`** | Read | **Not in official docs** | U13 in-process (Interop + runtime only) | R-10 controls |
| `IAttributeSink.GetAttribute(AttributeEnumType, string)` | Read | Documented | U13 in-process | Returned `Name` blank — ignore |
| `IAssemblyModel.GetAssemblyEntity(ICimEntity, AssInstance)` | Read | Documented | U7 in-process | Via existing highlighter |
| `ISelection.Selection` | Temp UI | Documented | U12 in-process | Via existing highlighter |
| `IAssemblyModel.GetBomManager()` → `IBOM.Export` | Writes temp CSV | Documented (2026) | U15 in-process (`cmNoFiltering`) | Option B/C only |

`IAssInstance.Attribute` is documented and installed as **Set-only** — never read. Candidate placement APIs (`GetRootInstance`, placed-instance `SubInstances`, `AssParentInstance` / `AssRootInstance`) and any active-document title/path API are **not yet approved CR-04 routes**; they must go through `AGENTS.md` §31 before use.

---

## 11. Read-only and safety

- `isReadOnly: true`, `FeatureKind.Validation`, `DocumentContextRequirement.Assembly`.
- Option A: zero filesystem writes. DO workbook opened read-only with `FileShare.ReadWrite` (existing reader).
- Option B/C: BOM CSV only under `%TEMP%\CimatronCustomization\CR04\<timestamp>\`; deleted in `finally`; never delete outside that folder; cleanup failure → `Warning` finding.
- Programmatic no-mutation evidence for every live run (document modified-state, model file timestamp/length; Option B/C adds model-folder listing guard).

## 12. Logging and data handling

The existing `ExecuteRegisteredFeature` logger records correlation/application metadata, canonical status, duration, `Findings=<total count>`, and the report message. CR-04 **must not claim separate logger fields** for DO rows, occurrences, evaluated components or per-status counts unless a platform logging change is separately approved. If operational counts are needed, place only non-sensitive aggregate counts in the feature outcome/report message. Never log component names, grades, attribute values, workbook content or full paths (`STEP-06-TABULAR-INPUT-CONTRACT.md`; `AGENTS.md` §12, §28).

---

## 13. Tests

### 13.1 Domain (≥ 95 % line / ≥ 90 % branch)

`SteelGradeNormalizerTests` — whitespace/NBSP/tab collapse; case-insensitivity; token mode; Strict match/mismatch; DO `1.208` → ambiguity Error; WerkstoffCanonical `1.208`≡`1.2080`, `1.2`≡`1.2000`; `P20`/`NAK80`/`H13` unchanged; absent never equal.
`ComponentKeyNormalizerTests` — normalization; no substring matching.
`SteelGradeDesignOrderEvaluationTests` — one test per §7.3 code; repeated placements / occurrence-collision inputs → one business finding with only safe refs; suppressed/excluded colliders still participate in reference-safety filtering; multi-body single and conflicting values; suppressed-only component → not found; component under suppressed sub-assembly → not found; duplicate DO key; empty DO; Option B blank Material → missing (not Error); Option C disagreement; aggregation precedence.

### 13.2 Application (≥ 90 %)

`VerifySteelGradeAgainstDesignOrderTests` (fake ports): each tabular stage failure → `Error`, CAD port not called; blank grade reaches Domain and follows D-16; CAD port exception → `Error`; mapping built from parameters; correct attribute names requested per `ComponentKeySource`; D-15 removes unsafe refs using full highlighter-scope `InstanceId` uniqueness while preserving the business finding status unless a verified placement route exists; C-03 (no `Passed` without comparison).
`DesignOrderInputPathResolverTests` — tokens, unknown token, invalid characters.
`SteelGradeDesignOrderValidationTests` — metadata (id, Validation, read-only, `CR-04`, Assembly), parameter definitions, traceability non-blank, Part context → `NotApplicable`.

### 13.3 Architecture

Domain has no Application/Interop/IO reference; DesignChecks depends only on Application/Domain; new adapters obey every §8.3 prohibition; 08B adapters unchanged (hash or source guard).

### 13.4 Fixtures (`tests/TestData/CR04/`)

Requirement-derived only, labelled *not customer layout*: matching set; mismatch; missing component; duplicate key; **key + blank grade**; numeric-cell `1.208`; text-cell `1.2080`; header with trailing space; Option B/C fixture may include blank `Material` and blank `Standard Number` rows.

### 13.5 Live Cimatron validation (user-run)

Known-answer assembly **`KA-ASM-CR04`** (disposable, derived from `KA-ASM-1`) includes a sub-assembly `SA-SLIDER-UNIT` **placed twice**, a nested occurrence under a suppressed parent, a top-level NG component, and dedicated bodies/attributes for the capability smoke checks.

| Component / case | Placement | Body attribute | DO row | Expected business result |
|---|---|---|---|---|
| `CORE_INSERT` | 1, nested single-placement control | `1.2738` | `1.2738` | Passed; nested highlight control |
| `CAVITY_INSERT` | 1, top-level | `1.2311` | `1.2738` | Failed; **top-level highlight must be visually confirmed** |
| `SLIDER` | inside repeated `SA-SLIDER-UNIT` | `1.2343` | `1.2343` | Passed business comparison; highlight behaviour governed by observed D-15 resolution, never first-match by assumption |
| `ID_COLLISION_ACTIVE` | evaluated occurrence whose local `InstanceId` collides with another active occurrence in the highlighter search scope | matching configured grade | matching DO grade | Business finding retained; **no normal highlight reference emitted** under D-15(a) |
| `ID_COLLISION_SUPPRESSED_CONTROL` | evaluated occurrence whose local `InstanceId` collides with an earlier suppressed occurrence in the highlighter search scope | matching configured grade | matching DO grade | Business finding retained; **no normal highlight reference emitted**; proves suppressed occurrences participate in the safety scope |
| `WEDGE` | 1 | requested grade attribute absent | `1.2379` | Smoke first records actual API behaviour; only then map to missing/other status |
| `WRONG_TYPE_CONTROL` | 1 | same requested name stored as non-string if Cimatron permits | matching text in DO | Smoke determines whether wrong type is distinguishable from absence |
| `HEEL_BLOCK` | 1, two bodies | `1.2738` / `1.2311` | `1.2738` | Failed — ambiguous |
| `GUIDE_STRIP` | 1 | `1.7131` | *(no row)* | Warning — not in DO |
| `SUPPORT_PILLAR` | 1 | *(none)* | *(no row)* | Not reported after missing-attribute semantics are verified |
| `LIFTER` | 1, suppressed | `1.2738` | `1.2738` | Failed — DO component not found |
| `NESTED_UNDER_SUPPRESSED_PARENT` | nested below suppressed sub-assembly | `1.2738` | present | Failed — DO component not found; proves inherited suppression |
| Assist-part control | as supported by KA model | configured | customer/test row per D-17 | Included/excluded exactly per D-17 and observed enumeration |
| *(none)* | — | — | `EJECTOR_PLATE` / `1.1730` | Failed — DO component not found |

**Slice 2A capability evidence** is recorded separately from the business Result Report: missing named attribute, wrong-type same-name attribute, repeated-placement identity/mapping, top-level highlight, suppressed-parent nesting, assist-part enumeration, and (if B/C) `cmNoFiltering` suppressed rows / blank `Standard Number`. If D-09(b) is selected, active assembly title/path retrieval is also verified here or in an earlier dedicated probe.

**Closure evidence:** clean quality gate (0 warnings / 0 errors, exit 0); Result Report matching the approved known answers; safe Highlight controls visually confirmed; programmatic no-mutation PASS; **every correlation ID recorded** including repeat/highlight runs; measured duration; one run on the customer's real DO sample and assembly convention. Option B/C adds BOM evidence with blank `Material` and suppression semantics.

---

## 14. Acceptance criteria

| ID | Criterion |
|---|---|
| AC-01 | With a valid DO sheet and `KA-ASM-CR04`, the Result Report reproduces the **approved** §13.5 business expectations after the Slice 2A capability gates are resolved. |
| AC-02 | Mismatch → `Failed`; match → `Passed`; a finding is highlightable only when its placement reference is proven safe. |
| AC-03 | Workbook/worksheet/header/read failures → `Error` with stable tabular code and no CAD evaluation. A key+blank-grade row is **not** rejected by generic required-field validation; it follows D-16. |
| AC-04 | Any unexpected Cimatron API/runtime failure → `Error`, never business `Failed`. |
| AC-05 | Part context → `NotApplicable`. |
| AC-06 | Missing/invalid parameter → `Error` naming it; no default used. |
| AC-07 | Programmatic no-mutation PASS; Option A writes no file. |
| AC-08 | Logs/report messages contain only approved aggregate counts; never names, grades, workbook contents or full paths. |
| AC-09 | Numeric-cell ambiguity follows approved D-05; no floating-point grade comparison. |
| AC-10 | D-15 is enforced for repeated placements and local-`InstanceId` collisions, including suppressed/excluded colliders: a normal reference is emitted only when `InstanceId` is unique across the full highlighter-equivalent traversal; no highlight silently targets an unintended occurrence. |
| AC-11 | Missing-name and wrong-type attribute semantics are runtime-evidenced before adapter status mapping is accepted. |
| AC-12 | Top-level highlight and inherited suppression are live-verified. |
| AC-13 | Assist parts follow D-17 and observed enumeration semantics. |
| AC-14 | If B/C: BOM is merged with traversal context; blank Material is data; suppressed-row semantics and D-18 are evidenced. |
| AC-15 | Coverage/static-analysis gates and architecture guards pass; 08B adapters unchanged. |
| AC-16 | Customer real-sample run reviewed and accepted (`AGENTS.md` §17 Gate 13). |
| AC-17 | ST06-DF-01/03/04/06 updated with evidence; ST06-DF-05 is not conflated with 08B helper cleanup; C-01/C-02 closed. |

---

## 15. Deferred-finding dispositions owned by CR-04

| ID | Resolution |
|---|---|
| ST06-DF-01 | D-10 — `ACCEPTED for CR-04`; CR-22 re-evaluates. |
| ST06-DF-02 | Not applicable (Text only); stays `DEFERRED`, mandatory for CR-22. |
| ST06-DF-03 | **Mandatory** — D-05 policy + tests + numeric-cell fixture + real-sample evidence. |
| ST06-DF-04 | D-11 from the real workbook; tested either way. |
| ST06-DF-05 | **Not owned by CR-04 unless packaging is changed.** DF-05 is the one-shot tabular runtime probe/client-packaging gate; it is not the 08B diagnostic/helper-duplication item. Track 08B helper retirement separately as technical debt. |
| ST06-DF-06 | Add a test proving a programming defect in the CR-04 path is not reported as `TABULAR_FILE_READ_ERROR`; narrow only with approval. |

---

## 16. Traceability record (`AGENTS.md` §14)

| Field | Value |
|---|---|
| Requirement ID | CR-04 |
| Business name | Steel Grade Design-Order Validation |
| Description | Compares each assembly component's steel grade (body-level string attribute, or BOM Material per D-01) with the expected grade in the Design-Order Input Sheet and reports matches, mismatches and coverage gaps. |
| Acceptance criteria | §14 AC-01…AC-17 |
| Inputs | Active Assembly; DO sheet from approved D-09 path strategy; §9 parameters |
| Outputs | Per-component findings with highlight references; aggregated outcome |
| Dependencies | assembly traversal/attribute port; optional BOM reader; optional D-09(b) assembly-identity port; `ReadMappedTabularData`; existing Result Report/highlighter for safe references |
| Cimatron APIs | §10 |
| Configuration | `STEEL-GRADE-DESIGN-ORDER-VALIDATION`, §9 |
| Failure modes | §7.3 stable message-prefixed codes; tabular errors; unexpected COM failures → Error; ambiguous/unproven highlight identity → business finding retained, unsafe reference omitted |
| Tests | §13 |
| Implementation version | Assigned at approval per `AGENTS.md` §19 (proposed 0.23.x) |
| Known limitations | §18, including `GetAllEntities()` undocumented in official SDK |

---

## 17. Implementation slices

| Slice | Content | Depends on | Live Cimatron |
|---|---|---|---|
| ~~0~~ | ~~Step 08B in-process confirmation~~ | **DONE — 0.22.2** | — |
| **1** — Domain rules | §7 pure types/tests; C-02 note; C-01 recorded; include D-16 blank-grade policy | D-04…D-08, D-14, D-16 | No |
| **2** — Application use case + ports | `IAssemblyBodyAttributeReader`, use case, path resolver; optional BOM port; optional D-09(b) assembly-identity port contract only after API-source plan; DF-01/04/06 decisions | Slice 1, D-01, D-09…D-11, D-18 if B/C | No |
| **2A — capability investigation (mandatory before adapter approval)** | **In-process Plugin diagnostic** using the real Result Report/highlight path for missing attribute, wrong type, repeated placement identity, deliberate `(InstanceId, ModelPid)` collision, top-level highlight, suppressed-parent nesting and assist parts; plus B/C suppression/key cases and D-09(b) identity API if selected. `KA-ASM-CR04` preparation is a separately approved modifying step with a recorded procedure (manual UI or governed setup utility). 2A gets its own file list, approval, evidence and 0.23.x package; any diagnostic launcher item is gated/removed before client packaging. | Prepared `KA-ASM-CR04`; D-02 attribute name. D-15/D-17 are outputs informed by 2A, not prerequisites. | **Yes** |
| **3** — Cimatron adapters | Implement only the routes proven/approved by 2A; architecture guards; version bump C-01 | Slice 2 + 2A results | Yes — adapter smoke |
| **4** — Feature + registration + config | `SteelGradeDesignOrderValidation`, `FeatureComposition`, production config block | Slices 1–3, D-03, D-12 | Yes |
| **5** — Validation & closure | §13.5 known-answer run, customer real-sample run, DF/C-item closure, docs | Slice 4, customer sample | Yes |

Slice 1 may begin once its business decisions are approved. **Slice 2A may run before or alongside Slice 2** as soon as `KA-ASM-CR04` and D-02 are ready, because the Slice-2 port contracts remain provisional until runtime identity/attribute semantics are known. Slice 3 may **not** be approved until Slice 2A closes the runtime assumptions. 2A receives its own governed package/version in the 0.23.x sequence; exact versions are assigned only at approval under `AGENTS.md` §19.

---

## 18. Risks and known limitations

| # | Risk / limitation | Mitigation |
|---|---|---|
| R-01 | Customer may record grade differently (body attribute vs Part material/BOM) | D-01 before adapter implementation |
| R-02 | Instance-level attribute is unreadable (U14) | Customer practice must place required attribute on body for A/C |
| R-03 | Real DO layout unknown | Real sample is acceptance prerequisite |
| R-04 | Numeric Excel cells can lose textual precision/trailing zeros | D-05; text format or separately approved raw-cell-type guard |
| R-05 | Model titles may differ from DO names | D-04 |
| R-06 | Large assemblies may make per-occurrence/body lookup expensive | Measure duration on customer assembly |
| ~~R-07~~ | ~~Single-placement nested highlight unverified~~ | Closed by 08B; does **not** close repeated-placement targeting |
| R-08 | Engraved text not verified | Out of scope |
| R-09 | `GetBomManager` E_FAIL observed historically | B/C → Error, no retry; A preferred unless customer practice requires BOM |
| R-10 | `IAssInstance.GetAllEntities()` absent from official SDK docs | Runtime re-verification per Cimatron version; known limitation |
| **R-11** | Repeated placement or local-`InstanceId` collision may make the first-match highlighter resolve the wrong occurrence or reject an otherwise unique `(InstanceId, ModelPid)` pair | D-15 + mandatory Slice 2A; emit a normal reference only when `InstanceId` is unique across the full unfiltered highlighter-equivalent traversal, unless placement-specific resolution is proven; measure collision frequency on a customer-like assembly |
| **R-12** | General adapter expands beyond the exact 08B known-answer call shape | Slice 2A + adapter smoke before feature wiring |
| **R-13** | Missing named attribute / wrong attribute type semantics are unknown | Dedicated smoke before mapping to `Absent` / `UnsupportedType` |
| **R-14** | B/C BOM rows lack placement/suppression/reference context and `cmNoFiltering` suppressed semantics are unknown | Always merge BOM with traversal; live B/C suppression test |
| **R-15** | D-09(b) needs assembly title/path, absent from current session snapshot | New verified identity port or choose D-09(a) |
| **R-16** | Assist parts may appear differently in top-level vs nested enumeration | D-17 + live enumeration evidence |
| **R-17** | `Finding` has no Code property and feature has no direct logger | Stable message prefixes; use existing executor/report logging unless separately approved platform change |

---

## 19. Customer input checklist (before Slice 3)

1. Real DO Input Sheet sample (anonymized is fine): worksheet name, header row, column headers.
2. Which DO column identifies the component, and how it relates to the Cimatron file/component name.
3. How designers record steel grade: a **body attribute** (give the exact name) or the **Part material property** (shows in BOM `Material`).
4. If attribute: confirmation it is on the **body** of every component, library parts included.
5. Whether grade values carry designations (`PCK`, `ESR`, hardness) and whether they must match.
6. Confirmation that numeric grades are always 4-digit Werkstoff numbers, or that the grade column is text-formatted.
7. Where DO files are stored relative to the tool assembly.
8. Which components are deliberately excluded from the DO (bought-out/standard).
9. Whether sub-assemblies (slider units, lifter units) are placed more than once in a tool, and whether per-placement highlight is required (D-15).
10. For library parts: does the BOM `Standard Number` equal the Cimatron model title, or the catalogue number?
11. What does a DO row with a component key but **blank steel grade** mean (invalid input, deliberate exclusion, or something else)?
12. Are Cimatron **assist parts** in scope for CR-04?
13. If BOM route B/C is desired: can BOM rows have blank `Standard Number`, and how should those rows be treated?
