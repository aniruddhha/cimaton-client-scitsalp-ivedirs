# Implementation Specification — CR-04 Steel Grade Verification Against Design Order

| Field | Value |
|---|---|
| Spec ID | IS-CR-04 |
| Spec version | **1.1** — aligned to platform 0.22.2 (supersedes 1.0, which was written against 0.21.4) |
| Requirement | CR-04 — Steel grade verification against DO (Design Order) |
| Business feature name | Steel Grade Design-Order Validation |
| Feature kind | `Validation` (read-only) — `CimatronCustomization.DesignChecks` |
| Platform baseline | `cimatron-customization` **0.22.2** (Phase 02, Step 08B CLOSED / USER VALIDATED). Production assembly metadata `0.22.0.0` (see C-01). |
| Authoritative requirement source | *Customer Requirement Document — Mold Design Checklist Automation in Cimatron* (Sheet 1), row CR-04 + §3 cross-cutting requirements |
| Supporting sources | Day-1 meeting transcript 05-Aug-2026 (00:12:32–00:16:06); `docs/platform/STEP-06-*`; `docs/platform/CIMATRON-2026-API-CAPABILITY-VERIFICATION.md` (U7, U8, U12, U13, U14, U15, U17); Step 08B closure evidence (0.22.0–0.22.2); official Cimatron API documentation via the `cimatron-api` MCP server |
| Spec status | **DRAFT — input to the implementing agent.** Not an implementation approval. The agent must follow `AGENTS.md` §33 (study → plan → file list → wait for explicit approval) for every slice. |
| Open decisions | 15 (Section 5). **BLOCKING** items must be answered before the corresponding slice is approved. |

### Changes from 1.0

| Area | Change |
|---|---|
| Capability gate | Step 08B is **done**: U7, U8, U12, U13 IN-PROCESS VERIFIED; U15 IN-PROCESS VERIFIED for the `cmNoFiltering` material read. Former Slice 0 is closed. |
| Port plan (§8) | Rewritten to build on the actual 0.22.2 ports (`IOccurrenceEntityAttributeReader`, `IAssemblyEntityMapper`, `IBomMaterialExporter`, nested-capable `IViewportEntityHighlighter`). Their signatures are known-answer-shaped (single model / single body / single BOM row), so CR-04 adds **general** sibling ports and leaves the verified 08B methods untouched. |
| BOM facts | BOM row key header is **`Standard Number`**; locked header contract `No.`, `Qty.`, `Standard Number`, `Sub Category`, `Category`, `Visible Part Size`, `Material`. Real BOMs contain rows with blank `Material` (cause of the 0.22.0 READ_ERROR) — this is normal data, not an error. |
| Decisions | D-01 now a genuine choice (both routes verified). D-04 gains option (c) BOM `Standard Number`. D-13 resolved. **New D-15** (occurrence-reference uniqueness under repeated / colliding sub-assembly instances). |
| Carry-forward | New §4 lists three items from the 08B review (version drift, undocumented second run, "Passed before confirmation" pattern). |
| Known-answer model | `KA-ASM-CR04` gains a **repeated sub-assembly** case. |
| Risks | R-07 closed; new R-11 (highlight reference ambiguity), R-12 (adapter generalization changes the verified call shape). |

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
| `ReadMappedTabularData` + `ExcelTabularDataReader` | Validated in-process (06E); CSV path additionally exercised in-process by 08B | DO sheet read; BOM CSV read (Option B/C) |
| `Finding`, `FindingAggregation`, `ResultStatus` | Validated | Result model |
| `IAssemblyStructureReader` | Validated | Hierarchy, suppression |
| **`IViewportEntityHighlighter` / `CimatronViewportEntityHighlighter`** | **Nested assembly references supported and IN-PROCESS VERIFIED (U12)** via the production Result Report Highlight | NG highlight of nested components — use as-is |
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

Verification was on `KA-ASM-1` (one nested level, one placement of the sub-assembly). Behaviour with **repeated sub-assembly placements, deeper nesting and large assemblies is not yet evidenced** — covered by D-15, R-11, R-12 and the §13.5 known-answer model.

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
| **D-01** | CAD-side evidence source | **A** body-level named string attribute (U13). **B** BOM `Material` column (U15; reflects the Part material property). **C** Both, cross-checked. | **A** — CRD-agreed mechanism, zero file writes, no `GetBomManager` E_FAIL exposure (seen once in 07C-4). **B** only if the customer confirms designers set the Part material property instead of an attribute. Both routes are now in-process verified, so this is purely a customer-practice question. | Customer + user | Slice 2, 3 |
| **D-02** | Attribute name and type | Name customer-defined; type `cmAttString` | Name = config parameter (no default). Type fixed to `cmAttString`. Other type → `Unsupported` finding. | Customer | Slice 3 config |
| **D-03** | DO Input Sheet layout | Worksheet, header row, key header, grade header | All config parameters; no invented defaults. Real customer sample required before acceptance. | Customer | Slice 4, acceptance |
| **D-04** | Row ↔ component key | **(a)** component `ModelTitle`. **(b)** a second configured string attribute on the body (e.g. item number). **(c)** BOM `Standard Number` (Option B/C only). | **(a)** unless the DO uses item numbers. Note: in 08B the BOM `Standard Number` equalled the model title (`KA-PART-1`); whether that holds for customer library parts must be confirmed before choosing (c). Matching = trimmed, whitespace-collapsed, case-insensitive ordinal. | Customer | Slice 1, 2 |
| **D-05** | Grade representation (**ST06-DF-03 mandatory**) | **Strict**; **WerkstoffCanonical** (pad `^\d\.\d{1,4}$` to 4 decimals both sides) | **Strict** + text-formatted grade column; DO value `^\d\.\d{1,3}$` → `Error DO_GRADE_AMBIGUOUS_NUMERIC`. WerkstoffCanonical only if customer confirms all numeric grades are 4-digit Werkstoff numbers. | Customer + user | Slice 1 |
| **D-06** | Suffix handling (`1.2738 PCK`) | Full string; grade token only | **Full string**; token mode as Boolean parameter. | Customer | Slice 1 |
| **D-07** | Normalization | Trim; collapse whitespace (incl. NBSP, tab); case-insensitive | All three, both sides; nothing else stripped. | User | Slice 1 |
| **D-08** | Coverage rules | §7.3 | DO row without component → `Failed`; attributed component not in DO → `Warning`; neither → not reported. | Customer + user | Slice 1 |
| **D-09** | DO file path supply | (a) absolute-path parameter; (b) path pattern `{AssemblyFolder}` / `{AssemblyTitle}`; (c) launcher file dialog (protected-file platform increment) | **(b)** first; (c) later if needed. | Customer + user | Slice 4 |
| **D-10** | Tabular error reporting (**ST06-DF-01**) | First-error; multi-error | First-error, `ACCEPTED for CR-04`; CR-22 re-evaluates. | User | Slice 2 |
| **D-11** | Header whitespace (**ST06-DF-04**) | Strict; trim | Decide from real workbook; trimming is a protected-file change to `TabularMappingEngine`/reader. | User | Slice 2 |
| **D-12** | Document context | Assembly only; Part also | **Assembly only.** | User | Slice 4 |
| ~~D-13~~ | ~~Nested highlight fallback~~ | — | **RESOLVED by 08B:** nested highlight is production-verified. Fallback `Unsupported` remains only for references that cannot be resolved (D-15). | — | — |
| **D-14** | Multi-body components; duplicate DO keys | §7.2 | 0 values → missing; 1 → value; >1 distinct → `Failed` ambiguous. Duplicate DO key → `Error`. | Customer + user | Slice 1 |
| **D-15** **(new)** | **Occurrence-reference uniqueness.** The production reference format is `entity|<instanceId>|<base64 model PID>|<entityId>`, and the highlighter resolves `<instanceId>` by **first match in a depth-first search**. Instance IDs are local to each assembly model, so (i) a sub-assembly placed twice yields identical references for both placements, and (ii) an ID in one sub-assembly may coincide with an ID elsewhere (the PID check guards only against different models). | **(a)** Accept: findings stay correct; highlight of a repeated placement shows the first match. **(b)** Detect ambiguity in the CR-04 adapter (same reference produced twice, or ID collision) and emit those references only once with a message that other placements exist. **(c)** Platform increment: path-based reference (`entity|<id path>|…`) in the highlighter (protected file, separate approval and live verification). | **(b)** for CR-04 — never highlight something that might be the wrong placement silently; findings list *all* placements by path in the message. **(c)** as a separate platform step if the customer needs per-placement highlight. | User | Slice 3 |

---

## 6. Functional behaviour

### 6.1 Run flow

```text
1. Pipeline pre-checks (existing): Cimatron available; active document = Assembly (else NotApplicable);
   parameters resolved/validated (missing required -> Error, no defaults).
2. Resolve DO file path (D-09). Missing file -> Error TABULAR_FILE_NOT_FOUND.
3. Read DO sheet via ReadMappedTabularData: ComponentKey (Text, required), SteelGrade (Text, required).
   Any tabular failure -> Error with stage + stable code; stop.
4. Validate DO rows (Domain): duplicate keys -> Error; D-05 ambiguity -> Error; zero rows -> Error; stop on Error.
5. Read CAD evidence through the D-01 route (§8). Any API/runtime failure -> Error (never Failed); stop.
   Option A: every occurrence (with suppression state incl. inherited), its model title, its bodies,
             per body the named-lookup result for <SteelGradeAttributeName> [and <ComponentKeyAttributeName>].
   Option B: cmNoFiltering BOM table -> rows {Standard Number, Material}; blank Material on a row is data.
6. Group by normalized component key; carry every non-suppressed placement's reference (subject to D-15).
7. Evaluate (Domain, pure) -> findings (§7.3).
8. Aggregate with FindingAggregation.
9. Result Report lists findings; Highlight uses carried current-session references.
```

### 6.2 Invariants

- **Read-only.** Option A writes nothing. Option B writes only the governed temp CSV (§11).
- **Error ≠ Failed.** COM/runtime failures, unreadable workbook, unresolvable entity → `Error` / `Unsupported`.
- **No invented defaults.** Every customer value comes from configuration (§9).
- **Suppression:** a suppressed occurrence, or any occurrence below a suppressed sub-assembly, is skipped and not counted. Hidden-but-not-suppressed occurrences are evaluated.
- **No persisted entity references.** References are current-session only.
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
| `DesignOrderSteelGradeEntry` | Key, original key text, grade, source row number |
| `CadComponentSteelGradeEvidence` | Key, original key, distinct grade values, body count, placement descriptions, `IReadOnlyList<ModelEntityReference>` |
| `SteelGradeDesignOrderEvaluation` | Pure `Evaluate(entries, evidence, policy)` → `IReadOnlyList<Finding>` |

### 7.2 Normalization and comparison rules

1. `null`/whitespace → *absent*; absent never equals anything (absent vs absent is not a match).
2. Trim; replace any whitespace run (space, tab, U+00A0) with one space.
3. `CompareGradeTokenOnly`: keep text before the first space.
4. **Strict:** key = step-3 text. **WerkstoffCanonical:** if `^\d\.\d{1,4}$`, right-pad decimals with `0` to 4 digits.
5. **DO-side guard (Strict only):** DO value matching `^\d\.\d{1,3}$` → `DO_GRADE_AMBIGUOUS_NUMERIC` (Error). Reason: numeric Excel cells lose trailing zeros through invariant text conversion (ST06-DF-03).
6. Equality ordinal-ignore-case; no fuzzy or substring matching.
7. Multi-body (D-14): collect from every `cmBody`; distinct non-absent values 0 → missing, 1 → value, >1 → ambiguous.
8. **Option B:** a BOM row with blank `Material` yields *absent* for that component; it is never an error by itself (lesson from 08B 0.22.0 → 0.22.1).

Grades are compared strictly as text; no floating-point comparison exists (`AGENTS.md` §25 satisfied).

### 7.3 Finding catalogue

`RequirementId = "CR-04"`; messages show original (non-normalized) text.

| Code | Condition | Status | Entity refs |
|---|---|---|---|
| `STEEL_GRADE_MATCH` | DO row matched; single CAD value equals DO grade | `Passed` | All placements (D-15) |
| `STEEL_GRADE_MISMATCH` | Single CAD value differs | `Failed` | All placements |
| `STEEL_GRADE_ATTRIBUTE_MISSING` | Matched component; no body carries the value (Option B: blank Material) | `Failed` | All placements |
| `STEEL_GRADE_ATTRIBUTE_AMBIGUOUS` | >1 distinct value across bodies | `Failed` | All placements |
| `DO_COMPONENT_NOT_FOUND` | DO key matches no non-suppressed occurrence | `Failed` | None |
| `CAD_COMPONENT_NOT_IN_DO` | Component carries a value but its key is not in the DO | `Warning` | All placements |
| `STEEL_GRADE_ATTRIBUTE_TYPE_UNSUPPORTED` | Named lookup returns a non-`cmAttString` type | `Unsupported` | All placements |
| `DO_DUPLICATE_COMPONENT_KEY` | Two DO rows normalize to one key | `Error` (stop) | None |
| `DO_GRADE_AMBIGUOUS_NUMERIC` | §7.2 rule 5 | `Error` (stop) | None |
| `NO_DO_ROWS` | Zero data rows | `Error` (stop) | None |

Option C (both sources) adds `STEEL_GRADE_SOURCE_DISAGREEMENT` (`Failed`) when attribute and BOM values differ for one component.

Outcome = `FindingAggregation.Aggregate(findings, ResultStatus.Passed)`; outcome message gives counts per status only (§12).

---

## 8. Architecture and file plan

Placement follows `docs/platform/CR-PLATFORM-GUIDE.md`. The agent presents the final file list for approval per slice.

### 8.1 Port strategy

The 0.22.2 ports are capability-named but their **signatures are known-answer-shaped**. CR-04 must not overload them with business meaning, and must not change the verified 08B methods (they back the 08B diagnostic until DF-05 removal). Therefore:

| Need | Approach |
|---|---|
| All occurrences × all bodies × named attribute (Option A) | **New general port** `IAssemblyBodyAttributeReader` (Application) + adapter `CimatronAssemblyBodyAttributeReader` (CimatronIntegration). Implementation follows the **same verified routines** as `CimatronOccurrenceEntityAttributeReader`: `GetInstances` traversal with depth guard 16 and sub-assembly recursion, `IAssInstance.GetAllEntities()` filtered to `cmBody`, `IAttributeSink.GetAttribute(cmAttString, requestedName)`, reference built in the production format `entity|<instanceId>|<base64 PID>|<entityId>`. |
| All BOM rows (Option B/C only) | **New general port** `IBomMaterialTableReader` + adapter `CimatronBomMaterialTableReader`, following `CimatronBomMaterialExporter`: `GetBomManager()`, `IBOM.Export(..., cmNoFiltering, ...)` to the governed temp folder, read through `ITabularDataReader` with the **locked header contract**, `Standard Number` required, `Material` optional per row, cleanup in `finally`. |
| Highlight | Existing `IViewportEntityHighlighter` **unchanged**. |
| Mapping to assembly context | Existing `IAssemblyEntityMapper` only if a component reference must be pre-mapped; the highlighter already maps nested references itself. |
| Mutation evidence | Same `MutationSnapshot` approach (document `IsModified`, file timestamp/length; Option B adds model-folder listing guard). |

**Code sharing decision (for plan approval):** the verified 08B adapters keep their private helpers. CR-04 introduces **one internal helper class** in CimatronIntegration (e.g. `AssemblyOccurrenceTraversal`, plus an internal reference-format helper) used by the new adapters. Refactoring the 08B adapters onto it is **not** part of CR-04 (avoids touching verified code); the duplication is recorded as technical debt to retire when the 08B diagnostic is removed under DF-05.

### 8.2 New port contracts

```csharp
// CR-04 (general): read named string attributes on every body of every assembly occurrence.
public interface IAssemblyBodyAttributeReader
{
    ModelReadResult<AssemblyBodyAttributeSnapshot> ReadNamedAttributes(
        IReadOnlyList<string> attributeNames);   // grade name [+ key name when D-04 = (b)]
}

// CR-04 Option B/C only: cmNoFiltering BOM rows (Standard Number, Material).
public interface IBomMaterialTableReader
{
    BomMaterialTableResult ReadMaterialTable();
}
```

`AssemblyBodyAttributeSnapshot` → list of `OccurrenceBodyAttributeEvidence`: `OccurrencePath` (instance-ID path from the root, e.g. `12/6`, for messages and D-15 detection), `InstanceId`, `ModelTitle`, `ModelPid`, `IsSuppressed` (own), `IsUnderSuppressedParent`, `Depth`, and per body `BodyNamedAttributeObservation` { `BodyEntityId`, `Reference` (`ModelEntityReference`), `AttributeName` = **requested** name, `NamedAttributeLookupStatus` (`Present` / `Absent` / `UnsupportedType`), `Value` }.

`BomMaterialTableResult` → `ResultStatus`, message, rows { `StandardNumber`, `Material` (may be empty) }, `NoMutationConfirmed`, `ModelFolderUnchanged`, `TempCleanupSucceeded`.

Returned `IAttribute.Name` must never populate `AttributeName`.

### 8.3 Adapter rules

Allowed (verified) routes only:

| Purpose | API route | Evidence |
|---|---|---|
| Traversal | `IAssemblyModel.GetInstances()`; recurse into sub-assembly models (`IAssInstance.Model` type `cmAssembly`); `SubInstances` presence check as in 08B; depth guard 16 | U8 in-process |
| Bodies | `IAssInstance.GetAllEntities()` → keep `cmBody` | U13 in-process (undocumented API — R-10) |
| Named attribute | `IAttributeSink.GetAttribute(cmAttString, name)`; success = non-null **and** type `cmAttString` | U13 in-process |
| Highlight reference | Production format consumed by `CimatronViewportEntityHighlighter` | U12 in-process |
| Suppression | `IAssInstance.IsSuppressed`; inherited from suppressed parents | Step 04 |
| BOM (B/C) | `IAssemblyModel.GetBomManager()`, `IBOM.Export(..., cmNoFiltering, ...)` | U15 in-process |

Forbidden (architecture-test enforced; extend the existing 07C-3/08B guards to the new adapters):

- `partModel.GetEntityById(...)` for occurrence bodies (`0x80040207` live failure);
- equality on returned `IAttribute.Name`;
- `IAssInstance.Attribute` getter (U14);
- any attribute setter, `Save`, or model-modifying call;
- `cmVisibleOnly` or any filter other than `cmNoFiltering`;
- `Material` declared required for every BOM row (0.22.0 defect);
- COM calls off the Cimatron UI thread (`AGENTS.md` §8).

"Attribute not present" is `Absent`, not an exception. Unexpected COM exceptions propagate to the pipeline → `Error`.

### 8.4 New files (expected)

| Layer | File(s) |
|---|---|
| Domain | `SteelGradeRepresentation.cs`, `SteelGradeComparisonPolicy.cs`, `SteelGradeNormalizer.cs`, `NormalizedSteelGrade.cs`, `ComponentKeyNormalizer.cs`, `DesignOrderSteelGradeEntry.cs`, `CadComponentSteelGradeEvidence.cs`, `SteelGradeDesignOrderEvaluation.cs` |
| Application | `IAssemblyBodyAttributeReader.cs`, `AssemblyBodyAttributeSnapshot.cs`, `OccurrenceBodyAttributeEvidence.cs`, `BodyNamedAttributeObservation.cs`, `NamedAttributeLookupStatus.cs`, `VerifySteelGradeAgainstDesignOrder.cs`, `DesignOrderInputPathResolver.cs`; Option B/C: `IBomMaterialTableReader.cs`, `BomMaterialTableResult.cs`, `BomMaterialRow.cs` |
| CimatronIntegration | `CimatronAssemblyBodyAttributeReader.cs`, internal `AssemblyOccurrenceTraversal.cs` (+ internal reference-format helper); Option B/C: `CimatronBomMaterialTableReader.cs` |
| DesignChecks | `SteelGradeDesignOrderValidation.cs` |
| Tests | Domain, Application, DesignChecks-feature and architecture tests (§13) |
| Docs | `docs/requirements/CR-04-STEEL-GRADE-DESIGN-ORDER-VALIDATION.md` (§16) |
| Test data | `tests/TestData/CR04/` fixtures |

### 8.5 Existing files expected to change (each needs explicit approval)

| File | Change |
|---|---|
| `src/CimatronCustomization.Plugin/FeatureComposition.cs` | Register the feature with its adapter(s) and the tabular reader |
| `config/CimatronCustomization.config.xml` | Add the `STEEL-GRADE-DESIGN-ORDER-VALIDATION` block (§9) |
| `tests/CimatronCustomization.ArchitectureTests/ArchitectureBoundaryTests.cs` | Extend guards to new adapters |
| `*.csproj` of changed projects | New file wiring (old-style project files) |
| `Properties/AssemblyInfo.cs` of every changed production assembly | Version bump (C-01) |
| `docs/platform/STEP-06-DEFERRED-FINDINGS.md` | DF-01/03/04/06 status |
| 08B handoff / `IMPLEMENTATION-HISTORY.md` | C-02 correlation note |
| `VERSION.md`, `CHANGELOG.md`, `IMPLEMENTATION-HISTORY.md`, `CHECKSUMS.sha256`, timesheet, session handoff | Standard package records |
| *Only if D-11 = trim:* `TabularMappingEngine.cs` / `ExcelTabularDataReader.cs` | DF-04 |
| *Only if D-15 = (c):* `CimatronViewportEntityHighlighter.cs` and reference producers | Separate platform increment, not in CR-04 slices |

Unchanged and must stay byte-identical: `CimatronOccurrenceEntityAttributeReader.cs`, `CimatronBomMaterialExporter.cs`, `CimatronAssemblyEntityMapper.cs`, `CimatronViewportEntityHighlighter.cs` (unless D-15 = c), `Step08BInProcessConfirmationFeature.cs`, and everything not listed.

---

## 9. Configuration

Feature id `STEEL-GRADE-DESIGN-ORDER-VALIDATION`. Examples are **placeholders**; no defaults are coded.

| Parameter | Type | Required | Example (illustrative) | Notes |
|---|---|---|---|---|
| `DesignOrderPathPattern` | String | Yes (D-09 b) | `{AssemblyFolder}\{AssemblyTitle}_DO.xlsx` | Tokens `{AssemblyFolder}`, `{AssemblyTitle}` only; unknown token → Error |
| `DesignOrderWorksheetName` | String | Yes | `Input Sheet` | Ignored for `.csv` |
| `DesignOrderHeaderRow` | Integer | Yes, min 1 | `1` | |
| `DesignOrderComponentKeyHeader` | String | Yes | `Component` | Exact header (D-11) |
| `DesignOrderSteelGradeHeader` | String | Yes | `Steel Grade` | |
| `SteelGradeSource` | String | Yes | `BodyAttribute` | Allowed: `BodyAttribute`, `BomMaterial`, `Both` (D-01) |
| `SteelGradeAttributeName` | String | Required unless `SteelGradeSource=BomMaterial` | `MATERIAL` | `cmAttString` on component bodies |
| `ComponentKeySource` | String | Yes | `ModelTitle` | Allowed: `ModelTitle`, `Attribute`, `BomStandardNumber` (D-04); `BomStandardNumber` only with `BomMaterial`/`Both` |
| `ComponentKeyAttributeName` | String | Required if `ComponentKeySource=Attribute` | `ITEM_NO` | |
| `SteelGradeRepresentation` | String | Yes | `Strict` | `Strict`, `WerkstoffCanonical` (D-05) |
| `CompareGradeTokenOnly` | Boolean | Yes | `false` | D-06 |

Cross-field inconsistencies and invalid enumerated text → `Error` naming the parameter and allowed values.

---

## 10. Cimatron API usage

| API | R/W | Official docs (MCP) | Platform status | Notes |
|---|---|---|---|---|
| `IAssemblyModel.GetInstances()` | Read | Documented | U8 in-process | Traversal |
| `IAssInstance.SubInstances` | Read | Documented as property `IAssInstance[]`; installed Interop: indexed `SubInstances[int,int]` → `Object` | U8 in-process | Follow installed signature (`get_SubInstances(1, 0)` as in 08B) |
| `IAssInstance.Model`, `IsSuppressed`, `Id` | Read | Documented | Validated | |
| **`IAssInstance.GetAllEntities()`** | Read | **Not in official docs** | U13 in-process (Interop + runtime only) | R-10 controls |
| `IAttributeSink.GetAttribute(AttributeEnumType, string)` | Read | Documented | U13 in-process | Returned `Name` blank — ignore |
| `IAssemblyModel.GetAssemblyEntity(ICimEntity, AssInstance)` | Read | Documented | U7 in-process | Via existing highlighter |
| `ISelection.Selection` | Temp UI | Documented | U12 in-process | Via existing highlighter |
| `IAssemblyModel.GetBomManager()` → `IBOM.Export` | Writes temp CSV | Documented (2026) | U15 in-process (`cmNoFiltering`) | Option B/C only |

`IAssInstance.Attribute` is documented and installed as **Set-only** — never read. No other API may be introduced without the `AGENTS.md` §31 workflow.

---

## 11. Read-only and safety

- `isReadOnly: true`, `FeatureKind.Validation`, `DocumentContextRequirement.Assembly`.
- Option A: zero filesystem writes. DO workbook opened read-only with `FileShare.ReadWrite` (existing reader).
- Option B/C: BOM CSV only under `%TEMP%\CimatronCustomization\CR04\<timestamp>\`; deleted in `finally`; never delete outside that folder; cleanup failure → `Warning` finding.
- Programmatic no-mutation evidence for every live run (document modified-state, model file timestamp/length; Option B/C adds model-folder listing guard).

## 12. Logging and data handling

Counts only through the existing pipeline logger: DO rows, occurrences, evaluated components, per-status finding counts, duration. Never log component names, grades, attribute values, workbook content or full paths (`STEP-06-TABULAR-INPUT-CONTRACT.md`; `AGENTS.md` §12, §28).

---

## 13. Tests

### 13.1 Domain (≥ 95 % line / ≥ 90 % branch)

`SteelGradeNormalizerTests` — whitespace/NBSP/tab collapse; case-insensitivity; token mode; Strict match/mismatch; DO `1.208` → ambiguity Error; WerkstoffCanonical `1.208`≡`1.2080`, `1.2`≡`1.2000`; `P20`/`NAK80`/`H13` unchanged; absent never equal.
`ComponentKeyNormalizerTests` — normalization; no substring matching.
`SteelGradeDesignOrderEvaluationTests` — one test per §7.3 code; repeated placements → one finding with all refs; multi-body single and conflicting values; suppressed-only component → not found; component under suppressed sub-assembly → not found; duplicate DO key; empty DO; Option B blank Material → missing (not Error); Option C disagreement; aggregation precedence.

### 13.2 Application (≥ 90 %)

`VerifySteelGradeAgainstDesignOrderTests` (fake ports): each tabular stage failure → `Error`, CAD port not called; CAD port `Unsupported`/exception → `Error`; mapping built from parameters; correct attribute names requested per `ComponentKeySource`; D-15(b) duplicate-reference handling; C-03 (no `Passed` without comparison).
`DesignOrderInputPathResolverTests` — tokens, unknown token, invalid characters.
`SteelGradeDesignOrderValidationTests` — metadata (id, Validation, read-only, `CR-04`, Assembly), parameter definitions, traceability non-blank, Part context → `NotApplicable`.

### 13.3 Architecture

Domain has no Application/Interop/IO reference; DesignChecks depends only on Application/Domain; new adapters obey every §8.3 prohibition; 08B adapters unchanged (hash or source guard).

### 13.4 Fixtures (`tests/TestData/CR04/`)

Requirement-derived only, labelled *not customer layout*: matching set; mismatch; missing component; duplicate key; numeric-cell `1.208`; text-cell `1.2080`; header with trailing space.

### 13.5 Live Cimatron validation (user-run)

Known-answer assembly **`KA-ASM-CR04`** (disposable, derived from `KA-ASM-1`), with a sub-assembly `SA-SLIDER-UNIT` **placed twice**:

| Component | Placement | Body attribute | DO row | Expected |
|---|---|---|---|---|
| `CORE_INSERT` | 1, nested | `1.2738` | `1.2738` | Passed |
| `CAVITY_INSERT` | 1 | `1.2311` | `1.2738` | Failed — mismatch |
| `SLIDER` | inside `SA-SLIDER-UNIT` → 2 placements via repeated sub-assembly | `1.2343` | `1.2343` | Passed; message lists both placement paths; highlight per D-15 |
| `WEDGE` | 1 | *(none)* | `1.2379` | Failed — attribute missing |
| `HEEL_BLOCK` | 1, two bodies | `1.2738` / `1.2311` | `1.2738` | Failed — ambiguous |
| `GUIDE_STRIP` | 1 | `1.7131` | *(no row)* | Warning — not in DO |
| `SUPPORT_PILLAR` | 1 | *(none)* | *(no row)* | Not reported |
| `LIFTER` | 1, **suppressed** | `1.2738` | `1.2738` | Failed — DO component not found |
| *(none)* | — | — | `EJECTOR_PLATE` / `1.1730` | Failed — DO component not found |

Required evidence: clean quality gate (0 warnings / 0 errors, exit 0); Result Report screenshot matching the table; Highlight of `CORE_INSERT` (nested) and `SLIDER` per D-15, visually confirmed; programmatic no-mutation PASS; **every correlation ID recorded** (C-02); measured run duration; one run on the **customer's real DO sample** and assembly convention. Option B/C adds a BOM run on the same model, including at least one row with blank Material.

---

## 14. Acceptance criteria

| ID | Criterion |
|---|---|
| AC-01 | With a valid DO sheet and `KA-ASM-CR04`, the Result Report reproduces §13.5 exactly. |
| AC-02 | Mismatch → `Failed` and highlightable; match → `Passed`. |
| AC-03 | Workbook/worksheet/column/blank-cell problems → `Error` with stable tabular code; no CAD evaluation. |
| AC-04 | Any Cimatron API/runtime failure → `Error`, never `Failed`. |
| AC-05 | Part context → `NotApplicable`. |
| AC-06 | Missing/invalid parameter → `Error` naming it; no default used. |
| AC-07 | Programmatic no-mutation PASS; Option A writes no file. |
| AC-08 | Logs contain counts only. |
| AC-09 | Numeric-cell ambiguity per approved D-05. |
| AC-10 | Repeated sub-assembly placements behave per approved D-15; no highlight ever silently targets an unintended placement. |
| AC-11 | Coverage/static-analysis gates and architecture guards pass; 08B adapters unchanged. |
| AC-12 | Customer real-sample run reviewed and accepted (`AGENTS.md` §17 Gate 13). |
| AC-13 | ST06-DF-01/03/04/06 updated with evidence; C-01 and C-02 closed. |

---

## 15. Deferred-finding dispositions owned by CR-04

| ID | Resolution |
|---|---|
| ST06-DF-01 | D-10 — `ACCEPTED for CR-04`; CR-22 re-evaluates. |
| ST06-DF-02 | Not applicable (Text only); stays `DEFERRED`, mandatory for CR-22. |
| ST06-DF-03 | **Mandatory** — D-05 policy + tests + numeric-cell fixture + real-sample evidence. |
| ST06-DF-04 | D-11 from the real workbook; tested either way. |
| ST06-DF-05 | Not owned by CR-04; note that the 08B diagnostic and the duplicated helpers (§8.1) retire together. |
| ST06-DF-06 | Add a test proving a programming defect in the CR-04 path is not reported as `TABULAR_FILE_READ_ERROR`; narrow only with approval. |

---

## 16. Traceability record (`AGENTS.md` §14)

| Field | Value |
|---|---|
| Requirement ID | CR-04 |
| Business name | Steel Grade Design-Order Validation |
| Description | Compares each assembly component's steel grade (body-level string attribute, or BOM Material per D-01) with the expected grade in the Design-Order Input Sheet and reports matches, mismatches and coverage gaps. |
| Acceptance criteria | §14 AC-01…AC-13 |
| Inputs | Active Assembly; DO sheet from `DesignOrderPathPattern`; §9 parameters |
| Outputs | Per-component findings with highlight references; aggregated outcome |
| Dependencies | `IAssemblyBodyAttributeReader` (and/or `IBomMaterialTableReader`), `ReadMappedTabularData`, `ITabularHeaderReader`/`ITabularDataReader`, `IViewportEntityHighlighter` |
| Cimatron APIs | §10 |
| Configuration | `STEEL-GRADE-DESIGN-ORDER-VALIDATION`, §9 |
| Failure modes | §7.3 Error/Unsupported codes; stable tabular codes; COM failures → Error |
| Tests | §13 |
| Implementation version | Assigned at approval per `AGENTS.md` §19 (proposed 0.23.x) |
| Known limitations | §18, including `GetAllEntities()` undocumented in official SDK |

---

## 17. Implementation slices

| Slice | Content | Depends on | Live Cimatron |
|---|---|---|---|
| ~~0~~ | ~~Step 08B in-process confirmation~~ | **DONE — 0.22.2** | — |
| **1** — Domain rules | §7 types + Domain tests; C-02 note; C-01 recorded | D-04…D-08, D-14 | No |
| **2** — Application use case + ports | `IAssemblyBodyAttributeReader` (+ `IBomMaterialTableReader` if D-01 = B/C) contracts, `VerifySteelGradeAgainstDesignOrder`, path resolver, Application tests; DF-01/04/06 decisions | Slice 1, D-01, D-09…D-11 | No |
| **3** — Cimatron adapters | `CimatronAssemblyBodyAttributeReader` (+ BOM table reader), internal traversal helper, architecture guards, CimatronIntegration version bump (C-01) | Slice 2, D-02, D-15 | Yes — adapter smoke on `KA-ASM-CR04` incl. repeated sub-assembly |
| **4** — Feature + registration + config | `SteelGradeDesignOrderValidation`, `FeatureComposition`, config block | Slices 1–3, D-03, D-12 | Yes |
| **5** — Validation & closure | §13.5 known-answer run, customer real-sample run, DF and C-item closure, docs | Slice 4, customer sample | Yes |

Slices 1–2 can start now; Slice 3 needs the customer answers in §19 (at minimum attribute name and D-01/D-04). Proposed package names: `phase-02-step-09-cr04-domain-rules-0.23.0.zip`, then PATCH/MINOR per approval.

---

## 18. Risks and known limitations

| # | Risk / limitation | Mitigation |
|---|---|---|
| R-01 | Customer may record grade differently (Part material vs body attribute) | D-01 with customer before Slice 3; both routes verified |
| R-02 | Attribute on the instance instead of the body is unreadable (U14) | Customer instruction: attribute on the **body**, library parts included |
| R-03 | Real DO layout unknown | Real sample is an acceptance prerequisite |
| R-04 | Numeric grade cells lose trailing zeros | D-05 guard (DF-03) |
| R-05 | Model titles may differ from DO names | D-04 options (b)/(c) |
| R-06 | Large assemblies: per-occurrence `GetAllEntities()` + per-body lookups | Measure duration on the customer assembly; record in evidence (`AGENTS.md` §28) |
| ~~R-07~~ | ~~Nested highlight unverified~~ | **Closed by 08B** |
| R-08 | Engraved text not verified | Out of scope |
| R-09 | `GetBomManager` E_FAIL (Option B/C) | `Error`, no automatic retry; Option A preferred |
| R-10 | `IAssInstance.GetAllEntities()` absent from official SDK docs | In-process verified (08B); Known-Limitation record; re-verify per Cimatron version; ask Cimatron to confirm support or name the documented alternative |
| **R-11** | **Highlight reference ambiguity**: production format resolves instance ID by first match; repeated sub-assembly placements share references; IDs are local per assembly model | D-15; repeated-sub-assembly case in `KA-ASM-CR04`; AC-10 |
| **R-12** | **Generalized adapter is not the exact verified call shape** (08B verified one model / one body ID / one BOM row) | Same API routes only (§8.3); Slice 3 live smoke run on `KA-ASM-CR04` before Slice 4 |

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
