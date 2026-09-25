# Phase 02 — CR Platform Extension Specification

**Project:** Cimatron Customization
**Baseline:** `0.10.0` (Phase 01 Step 10 closed; see `docs/history/session-handoffs/phase-01-step-10-to-phase-02-cr-implementation-handoff.md`)
**Approach:** Platform-first. Build the shared capabilities every customer CR needs **before** implementing any CR business rule.
**Spec type:** Requirements and design constraints only. This document contains **no implementation logic and no code**. Class and interface names are *suggested*; the implementing agent may propose better names in its plan, subject to approval.
**Authority:** `AGENTS.md` overrides this spec wherever they conflict. Any conflict must be reported to the user, not silently resolved.

---

## 0. Instructions to the implementing agent

1. Read `AGENTS.md` completely. Then read the Phase 01 → Phase 02 handoff file, `VERSION.md`, `IMPLEMENTATION-HISTORY.md` and `CHANGELOG.md`.
2. Read this spec completely before planning any step.
3. Implement **one step at a time** (Section 5). Each step is a separate approval cycle and a separate progressive ZIP (AGENTS §18, §19).
4. For each step, present the AGENTS §18 plan first:
   - scope and new files;
   - every existing file that must change, with the reason, impact and regression risk;
   - the API assumptions to verify;
   - the tests to add;
   - the live Cimatron validation steps and expected results.

   Then **wait for explicit approval**.
5. Do not implement any customer CR rule (CR-01…CR-27) or SA automation in these steps. The platform is proven only with internal/foundation features (Section 4.10).
6. Every Cimatron API usage must be verified before use (Section 3). If verification is impossible in-session, mark the item `UNVERIFIED`. Then propose a verification method the user can run: object browser on the interop DLLs, Journaling, or a probe feature. Do not guess signatures.
7. The user runs all builds, tests, quality gates and Cimatron validation, and supplies the evidence (AGENTS §17A).
8. The first package must also close the documentation lag noted in handoff §9. That means recording the Step 10 PASS evidence in `VERSION.md`, `CHANGELOG.md` and `IMPLEMENTATION-HISTORY.md` without rewriting history.

---

## 1. Purpose and principle

The customer requirement documents define 19 checklist items to be built through the Cimatron API:

> CR-01, 02, 03, 04, 06, 08, 09, 10, 16, 18, 19, 20, 21, 22, 23, 24, 25, 26, 27

The 0.10.0 baseline was proven with one trivial feature. It lacks capabilities that **every** one of these CRs needs.

**Governing rule:** every platform capability in this spec must trace to at least one CR (Section 7). Nothing speculative is built. If the agent believes an additional platform capability is needed, it must name the CR that requires it and request approval.

**Authoritative requirement sources (supplied by the user separately):**
- *Customer Requirement Document — Mold Design Checklist Automation (Sheet 1)*: CR-01…CR-27.
- *Customer Requirement Document — Mold Design Software Automation (Sheet 2)*: SA-01…SA-15. Informational only in this spec; SA items are not in scope.

Cross-cutting customer requirements this platform must enable (from the Sheet 1 document §3):
- **One-click execution** of each check.
- **Highlight non-conforming (NG) geometry.**
- **Automatic zoom** to NG locations on large molds.
- **User-editable rule values**, never hard-coded.
- **Attribute-based component identification.**
- **Color-code inputs.**
- **Excel Input Sheet ("DO") as master data.**

---

## 2. Scope

### In scope
Platform capabilities WP1–WP10 (Section 4), delivered in Steps 01–06 (Section 5).

### Out of scope
- Any CR/SA business rule, threshold logic or customer-specific recognition logic.
- Specialized geometry analyzers: hole-chain recognition, draft evaluation, minimum distance, cooling-circuit traversal, weight calculation. These belong to later CR group steps. This spec only reserves their port names (Section 4.5.3).
- Modifying automations (Automations project content). The platform stays read-only in these steps.
- The analyzer warning cleanup listed in handoff §7.
- SonarQube server validation.
- Installer/client packaging changes, except where a new runtime dependency (WP8) requires a documented client-package impact.

---

## 3. Cimatron API facts and verification register

### 3.1 Verified facts (usable as design constraints)

| # | Fact | Source |
|---|---|---|
| V2 | The Phase 01 baseline successfully registered `CimatronPluginCommand : ICimApiCommandPlugin` through Cimatron's external-command registration mechanism and persisted the registration in `ExternalCommands.ini`. The validated runtime evidence confirms that the INI registration key identifies the plugin class rather than the `ICimWpfCommand` execution class. | Phase 01 live validation / handoff evidence on Cimatron 2026.0 SP2P1 |
| V3 | Cimatron 2026 provides `IOpenGLService` to overlay custom graphical content in the viewport. It also provides `ISetsFolder` (programmatic set folders) and `IBOM` (assembly BOM dataset). | `https://api.cimatron.com/assets/docs/release_notes/2026/new_features.htm` |
| V4 | `IApplication` exposes (among others): `GetActiveDoc`, `GetAttributeFactory`, `GetDI`, `GetActiveViewOpenGlService`, `GetExternalPaneManager`, `GetPreferenceManager`, `CreateContext`, `GetPoolCommands`. Existence is confirmed; per-method semantics must still be verified before use. | `https://api.cimatron.com/assets/docs/cimatron_e_api/elite_api_interfaces/iapplication/iapplication.htm` |
| V5 | `CimApplicationProvider.GetApplication()` works only from a class-library project. The baseline already uses it. | `https://api.cimatron.com/assets/docs/cimatron_e_api/topic_documents/object_containers/cimapplicationprovider.htm` |
| V6 | AGENTS §8: for Cimatron 2026+, prefer the documented topology interfaces `IBody`, `IFace`, `IEdge`, `IVertex`. Entity IDs must not be assumed persistent. | `AGENTS.md` |
| V7 | Supported runtime observed by the user: Cimatron 2026.0 SP2P1 (`2026,0201,2064,662`). | Handoff §3, §5 |

**Note on source authority:** use the official Cimatron SDK topic pages under `https://api.cimatron.com/assets/docs/...`, the installed target-version interop DLLs/object browser, and verified runtime behaviour as primary evidence in accordance with `AGENTS.md`. Community or GitHub mirrors may be used only as discovery aids and must not be promoted to verified API facts without confirmation from an authoritative source.

**Note on the doc portal:** `https://api.cimatron.com/Cimatron_SDK.htm` is a JavaScript-driven RoboHelp shell. Fetch individual topic pages under `https://api.cimatron.com/assets/docs/...` directly where available.

### 3.2 Items requiring verification before use

The earlier claim that Cimatron supports only one `ICimApiCommandPlugin` class per DLL is **not verified**. It must not be used as an architectural constraint. The product preference for a single launcher remains independent of that technical question and is governed by D1; U11 verifies the runtime capability.

| # | Needed capability | Candidate API area | Needed by | Verification method |
|---|---|---|---|---|
| U1 | Enumerate assembly components/instances, names, hierarchy, instance placement | Assembly document / instance interfaces | WP5 | SDK docs + object browser + probe feature |
| U2 | Read component/entity attributes (name → value) | `GetAttributeFactory` / attribute interfaces | WP5 | SDK docs + probe on a customer-style attributed component |
| U3 | Read entity/face color | Entity/face display properties | WP5 | SDK docs + probe |
| U4 | Enumerate bodies/faces; face type, normal, area, bounding box | `IBody`/`IFace` topology | WP5 | SDK docs + probe |
| U5 | Non-persistent highlight of entities or locations | `IOpenGLService` overlay **or** a selection/highlight mechanism | WP7 | SDK docs + live test proving no model modification |
| U6 | Zoom/fit the active view to a region or entity | View / display interfaces (`GetDI`?) | WP7 | SDK docs + live test |
| U7 | Interactive user picks (entity, face, direction) | Selection/pick interfaces | WP7 | SDK docs + Journaling |
| U8 | Detect whether the document was modified (to prove read-only behaviour) | Document state property | WP7, all read-only checks | SDK docs + live test |
| U9 | Docked pane hosting, if the later docked-launcher variant is approved | `IExternalPane` / `GetExternalPaneManager` | WP2 | SDK docs + registration test |
| U10 | Threading rules for UI windows and COM calls in-process | General API rules | WP2, WP7 | SDK docs; default to UI-thread-only COM access |
| U11 | Whether one DLL can expose and register multiple `ICimApiCommandPlugin` classes/commands in Cimatron 2026 | Plugin command registration | WP2 / D1 | Official SDK + object browser if relevant + controlled live probe with two plugin classes in one DLL |

For every U-item the step plan must state the verified interface/method or registration behaviour, the source URL or DLL/runtime evidence, the Cimatron version, and any version-specific behaviour (AGENTS §8). If an item cannot be verified, the step delivers a controlled `Unsupported` result for that capability rather than a guess. For U11, verification must be a controlled probe and must not silently alter the production baseline.

---

## 4. Work packages

Each work package lists **what** must exist and the **rules** it must obey, never how to implement it.

### 4.1 WP1 — Composition and architecture boundary

**Problem (verified in the baseline):** `ArchitectureBoundaryTests` forbids Plugin from referencing DesignChecks and Automations. Plugin is the composition root that builds the `FeatureRegistry`. So a feature placed in DesignChecks cannot be registered.

**Requirements**
- Allow `Plugin → DesignChecks` and `Plugin → Automations` project references, for composition only. Plugin still must not contain feature logic.
- Update the architecture tests to encode the new allowed graph explicitly. Add a rule that DesignChecks and Automations do not reference each other, and neither references interop or CimatronIntegration (already partially enforced).
- Composition stays explicit: no reflection discovery and no DI framework (handoff §2).
- Move feature composition out of `CimatronCommandExecution.OnCommand` into a dedicated composition class so the command handler stays thin.

**Suggested names:** `FeatureComposition` (Plugin); `ArchitectureBoundaryTests` (modified).

### 4.2 WP2 — Feature launcher and command UX

**Design preference:** Use one launcher command as the default product UX because it scales cleanly as the number of CR/SA features grows. This is a product-design preference, not a verified Cimatron API constraint. U11 must settle whether multiple plugin command classes from one DLL are technically supported.

**Requirements**
- The preferred default is one command that opens a **Feature Launcher** listing all registered features.
  - Group them by kind (Validation/Automation) and by the platform-owned functional taxonomy defined in §5, pending user approval of that taxonomy.
  - Show the business name, requirement IDs and applicability to the current document context.
- The user can run a single feature, and optionally a selected set of features in sequence ("run checklist"). Each run produces its own report under one correlation ID.
- Features not applicable to the current context are shown but disabled, with the reason.
- Startup diagnostics remain available from the launcher (not a mandatory modal on every run).
- The launcher must not call Cimatron COM from background threads (U10).

**Decision D1:** command/launcher UX:
- **Option A (recommended):** one registered command opens a modal/modeless launcher window.
- **Option B:** one registered command per check/feature, if U11 proves that multiple command classes from one DLL are supported and the user prefers that UX.
- A docked `IExternalPane` remains a later launcher-hosting variant and requires U9.

Recommendation: use the single launcher window first even if U11 proves multiple commands are technically possible, because the launcher scales better as the feature count grows.

**Suggested names:** `FeatureLauncherWindow` (Plugin UI); `ListAvailableFeatures` (Application use case); `FeatureAvailability` (Application).

### 4.3 WP3 — Feature execution contract and findings model

**Problem:** `IApplicationFeature.Execute()` takes no input and returns only status plus message. CRs must report *which* entities failed, with measured value and limit, and must receive parameters.

**Requirements**
- **Execution input.** Features receive an execution context containing:
  - correlation ID;
  - the feature's validated rule parameters (WP4);
  - optional user-provided run inputs (WP7 picks/directions);
  - access to model-reading ports only through Application interfaces.
- **Findings.** A feature outcome carries zero or more findings. Each finding holds:
  - rule/requirement ID;
  - result state from a **canonical result-state enum**; free-text status strings are not permitted;
  - the enum must conform to the approved AGENTS §10 state model, subject to Decision D6 for Automation success semantics;
  - human-readable message;
  - optional measured value and limit, with units;
  - optional entity reference(s);
  - optional 3D location for zoom.
- **Aggregation.** The overall feature status is derived from its findings by a documented, tested rule in Domain. API/runtime failures remain `Error` and are never converted to `Failed` (AGENTS §10).
- **Canonical status migration.** Step 01 replaces any free-text execution status with the canonical enum. The existing `Succeeded` wording must not remain as an ad-hoc string. Its Automation semantics are resolved explicitly by D6 rather than silently deleted or retained.
- **Units and tolerance.** Domain value types carry explicit units and tolerance-aware comparison. Exact floating-point equality is forbidden (AGENTS §25).
- **Entity references.** A Domain-side opaque reference, valid for the current session only (V6). Resolution back to Cimatron objects happens only in CimatronIntegration.
- **Feature metadata.** `FeatureDefinition` gains:
  - requirement IDs (list; one feature may cover several CRs, e.g. CR-19 and CR-20);
  - functional group;
  - required document context (Part / Assembly / either);
  - read-only flag. It must be true for Validation features and represents declared intent. Architectural enforcement comes from exposing only read-only Application/CimatronIntegration ports to Validation features; runtime proof comes from the live "document not modified" validation.
- **Logging.** Populate the existing `ApplicationLogEntry.RequirementId` field, currently always empty, from feature metadata. Log a findings count summary, not proprietary geometry (AGENTS §12).
- **Migration.** The existing `ActivePartReadinessFeature` migrates to the new contract with no behaviour change. Its existing tests must still pass unchanged or with justified, approved edits.

**Suggested names:**
- Application: `FeatureExecutionContext`, `FeatureRunInputs`, `IApplicationFeature` (modified), `FeatureDefinition` (modified), `DocumentContextRequirement`.
- Domain: `Finding`, `FindingCollection`, `FindingAggregation`, `ModelEntityReference`, `Measurement`, `Length`, `Angle`, `Point3`, `Direction3`, `GeometryTolerance`.

### 4.4 WP4 — Versioned rule-parameter configuration

**Problem:** config schema 1 holds only logging settings. CRs need user-editable values, for example:
- the 1:5 ratio (CR-01);
- 3° (CR-02);
- Ø100 mm periphery (CR-03);
- 3×D (CR-10);
- +2° (CR-24);
- 200 mm (CR-25);
- material ratios and color codes (CR-06, CR-18–20).

**Requirements**
- Introduce **config schema 2** with per-feature parameter sets. Schema 1 files must load through a documented, tested migration (AGENTS §13), and logging settings keep working.
- Each parameter definition declares its name, type, unit, allowed range and whether it is required. Parameter *values* for real CRs are added by the CR steps, not here.
- **No silent unsafe defaults** (AGENTS §13). A missing or invalid required parameter yields a controlled `Error` report naming the parameter and the config file. It never falls back to an invented value.
- Parameter values are readable in the launcher (per feature).
- Color-code parameters use a documented representation. Its mapping to Cimatron color values depends on U3.

**Decision D2:** parameter editing in the launcher UI (write back to config) vs file-only editing in Step 02. Recommendation: file-only plus read-only display in Step 02. In-UI editing can come as a later step if the customer asks for it.

**Suggested names:** `RuleParameterDefinition`, `RuleParameterSet`, `IRuleParameterProvider` (Application); `XmlRuleParameterProvider`, `ConfigurationSchemaMigration` (Infrastructure).

### 4.5 WP5 — Cimatron model access ports (read-only)

**Problem:** CimatronIntegration exposes only the document type. It references only `interop.CimatronE` and `interop.CimServicesAPI`.

#### 4.5.1 Ports to define in Application and implement in CimatronIntegration (this spec)

| Port (suggested) | Provides | Needed by |
|---|---|---|
| `IAssemblyStructureReader` | Component tree, instance names, hierarchy, placement, part/assembly kind | CR-03, 06, 08, 19, 20, 22–26 |
| `IEntityAttributeReader` | Attribute name → value for components/entities | CR-04, 22, 26 (customer agreed to attribute-based identification) |
| `IEntityColorReader` | Color of faces/entities | CR-06, 18, 19, 20 |
| `IBodyTopologyReader` | Bodies, faces, face type, face normal at a point, area, bounding box | CR-01, 02, 09, 10, 16, 21 and later analyzers |

#### 4.5.2 Rules
- Read-only. No port may modify, move, save or recolor anything (AGENTS §27).
- Unsupported geometry/entity types yield a controlled `Unsupported` outcome and are logged (AGENTS §8).
- Minimize COM round-trips and avoid repeated full-model scans (AGENTS §28). A per-run read cache inside CimatronIntegration is allowed. It must not outlive a single feature run.
- Add interop references only as needed (for example `interop.CimBaseAPI`, `interop.CimMdlrAPI`), each justified in the plan and verified against the installed Cimatron.
- Integration tests follow the existing `CimatronIntegrationTests` pattern. Real-Cimatron validation cannot be replaced by mocks (AGENTS §9).

#### 4.5.3 Reserved names (define later in CR group steps, not now)
- `IHoleChainReader`: CR-06, 08, 09, 10, 16.
- `IDraftEvaluator`: CR-02, 21.
- `IMinimumDistanceEvaluator`: CR-01, 03.
- `ICoolingCircuitReader`: CR-06, 08.
- `IMassPropertiesReader`: CR-22.

### 4.6 WP6 — Document-context applicability

**Requirements**
- The execution pipeline checks each feature's declared document-context requirement before executing it. The result is `NotApplicable` (wrong context) or `Error` (no application/document), never an exception surfaced as a crash.
- Assembly context must be first-class. Most mold checks run on the full assembly (CR-03, 06, 08, 19, 20, 26).
- The existing `CimatronSessionSnapshot` / `CimatronDocumentContext` are reused; extend them rather than duplicate them.

### 4.7 WP7 — Result presentation: report, highlight, zoom, picks

**Requirements**
- **Result report window:**
  - lists the findings per feature: status, message, measured value, limit, requirement ID;
  - allows sorting and filtering by status;
  - shows the summary counts.
- **Highlight:**
  - selecting a finding highlights its entities or location in the viewport;
  - "highlight all NG" is available;
  - highlighting must be **non-persistent**: it must not change entity colors, create features or mark the document modified;
  - it is cleared when the report closes or a new run starts.
- **Zoom:** selecting a finding zooms the view to it. This is the customer requirement from CR-21, applied generally.
- **Picks:** a feature can request user inputs before or during the run (entity pick, face pick, direction). CR-21 needs a direction per slider. The pick request is declared in feature metadata so the launcher can collect it.
- **Proof of read-only:** a live validation step demonstrates that after running a check with highlighting, the document is not marked modified (U8).

**Decision D3:** highlight mechanism, chosen after U5 is verified:
- `IOpenGLService` overlay (verified to exist, 2026); or
- a selection-based highlight; or
- both.

Recommendation: prefer the overlay if U5/U8 show it is fully non-persistent.

**Suggested names:**
- Application: `IFindingPresenter`, `IFindingHighlighter`, `IViewNavigator`, `IUserInputRequester`.
- Plugin/CimatronIntegration: `FindingReportWindow`, `ViewportFindingHighlighter`, `CimatronViewNavigator`, `CimatronUserInputRequester`.

### 4.8 WP8 — Tabular (Excel) input

**Problem:** CR-04 (steel grade vs DO Input Sheet) and CR-22 (finger-pin diameter vs weight table) need customer Excel data. There is no reader today.

**Requirements**
- **Port.** An Application port reads a worksheet as a table of typed rows. Sheet, header row and column mapping come from configuration (WP4), because the customer layout is not yet known.
- **Dependency.** Choose a pinned `.xlsx` reader compatible with .NET Framework 4.8 and approve it through `DEPENDENCIES.md` (license, version, source).
  - This is a **runtime client dependency**, unlike the existing dev-only tools. The plan must state the client-package impact (AGENTS §23).
  - Candidates for the agent to evaluate (not preselected): DocumentFormat.OpenXml, ExcelDataReader, ClosedXML.
- **Failure modes.** A missing file, locked file, missing sheet/column or unparsable value must give an actionable `Error`, never a partial silent read.
- **Data handling.** Customer file content is not logged beyond counts and identifiers (AGENTS §12, §28).

**Decision D4:** library choice. Alternatively, the customer supplies CSV exports and no dependency is added.

**Suggested names:** `ITabularDataReader`, `TabularTable`, `TabularRow`, `TabularSourceDefinition` (Application); `XlsxTabularDataReader` (Infrastructure).

### 4.9 WP9 — Traceability and documentation

- Each feature definition carries the AGENTS §14 traceability fields. For platform steps these fields describe the platform capability; customer CR fields are filled in by the CR steps.
- `IMPLEMENTATION-HISTORY.md`, `CHANGELOG.md` and `VERSION.md` are updated per step; history is not rewritten.
- A new doc, `docs/platform/CR-PLATFORM-GUIDE.md`, explains how a future CR is added:
  - where its files go;
  - how it is registered;
  - how parameters are declared;
  - how findings are returned;
  - which ports to use.

  This is the "same platform for future CRs" contract.

### 4.10 WP10 — Internal probe features (platform proof, no customer logic)

Following the Phase 01 precedent (`Active Part Readiness Check`), each platform step is proven live with an internal foundation feature that contains no customer rule.

| Probe (suggested) | Proves | Also yields |
|---|---|---|
| `ModelInspectionProbe` | WP3, WP5, WP6: reports assembly components, their attributes and colors as `Passed` informational findings | Discovery data on how the customer's real models carry attributes and colors, which the CR steps need |
| `FindingPresentationProbe` | WP7: emits synthetic findings on picked faces to exercise report, highlight, zoom, picks and the read-only proof | — |
| `RuleParameterProbe` | WP4: reads a declared test parameter set and reports validation results | — |
| `TabularInputProbe` | WP8: reads a configured sample sheet and reports row/column counts | — |

**Rules**
- Probes are foundation/internal features with IDs prefixed `FOUNDATION-`, like the existing readiness feature.
- Probes must be hideable from customer builds by configuration.

**Decision D5:** whether probes ship to the client (hidden) or exist only in dev builds.

---

## 5. Step plan (one step = one approval = one ZIP)

Versions are proposals. The agent applies AGENTS §19 rules and justifies the bump in each plan.

| Step | Name (ZIP step-name) | Work packages | Proven by | Proposed version |
|---|---|---|---|---|
| 01 | `feature-contract-and-composition` | WP1, WP3, WP6, WP9 (+ handoff §9 doc closeout) | Readiness feature migrated; unit and architecture tests | 0.11.0 |
| 02 | `rule-parameter-configuration` | WP4 | `RuleParameterProbe`; schema 1→2 migration tests | 0.12.0 |
| 03 | `feature-launcher-and-report` | WP2, WP7 (report window only) | Launcher lists and runs features; report shows findings | 0.13.0 |
| 04 | `cimatron-model-access-ports` | WP5 | `ModelInspectionProbe` on a real mold assembly | 0.14.0 |
| 05 | `viewport-highlight-zoom-picks` | WP7 (highlight, zoom, picks, read-only proof) | `FindingPresentationProbe` | 0.15.0 |
| 06 | `tabular-input-reader` | WP8 | `TabularInputProbe` on a sample sheet | 0.16.0 |

**Ordering rationale**
- 01 unblocks everything.
- 02 and 03 are pure platform with no new COM surface.
- 04 introduces model reads.
- 05 depends on 04's entity references.
- 06 is independent. It can move earlier if D4 is decided early.

**After Step 06**, CR implementation begins in the following **platform-owned proposed functional groups**:
- attribute-based: CR-04, 26, 27;
- hole and cooling: CR-06, 08, 09, 10, 16;
- draft: CR-02, 21;
- slider rules: CR-22–25;
- vents: CR-18–20;
- geometry: CR-01, 03.

These groups are an implementation/UX taxonomy proposed by this specification. They are **not** attributed to the customer's Sheet 1 Section 5. The taxonomy must be user-approved before it becomes governing product grouping.

Each group step first defines the reserved analyzer ports it needs (Section 4.5.3).

---

## 6. Test and validation requirements (all steps)

- **TDD** is the default (AGENTS §9). Domain and Application logic are covered by unit tests. Existing coverage thresholds must be preserved (Domain ≥90%, Application ≥90%, Infrastructure ≥80%, per the Step 10 evidence).
- **Architecture tests** are updated only where this spec explicitly changes the dependency graph (WP1). Every other boundary rule stays.
- **Regression:** existing tests keep passing. Existing files outside the approved scope stay byte-identical (handoff §2).
- **Live Cimatron validation** per step, with explicit expected results written in the plan:
  - startup still works;
  - the readiness feature still passes on a Part;
  - the step's probe produces its expected output;
  - no crash or second Cimatron process appears;
  - structured log entries share one correlation ID and carry requirement IDs.
- **Read-only proof** (Step 05, and later every Validation CR): the document is not marked modified after a run.
- **Performance:** the plan states a measurable expectation for the model-read probe on a customer-size assembly (AGENTS §28). The customer is used to about 15 seconds for the existing cooling analysis (CR-05), which is a reasonable reference.

---

## 7. Traceability — platform capability → CRs

| Capability | CRs requiring it |
|---|---|
| WP1 composition boundary | All 19 |
| WP2 feature launcher / command UX | All 19 (one-click execution; single-launcher preference, with U11 verifying multi-command capability) |
| WP3 findings, units, entity refs | All 19; measured-vs-limit: CR-01, 02, 03, 10, 18, 22, 24, 25 |
| WP4 rule parameters | CR-01, 02, 03, 06, 10, 18, 19, 20, 22, 24, 25 |
| WP5 assembly structure | CR-03, 06, 08, 19, 20, 22–26 |
| WP5 attributes | CR-04, 22, 26 |
| WP5 colors | CR-06, 18, 19, 20 |
| WP5 topology | CR-01, 02, 09, 10, 16, 21 |
| WP6 assembly context | CR-03, 06, 08, 19, 20, 26 |
| WP7 highlight / zoom | All 19 (highlight NG); auto-zoom: CR-21 |
| WP7 picks / directions | CR-21 (slider direction), CR-03 (drop points, if not recognizable) |
| WP8 tabular input | CR-04, 22 |

---

## 8. Decisions required from the user

| # | Decision | Recommendation | Needed before |
|---|---|---|---|
| D1 | Command/launcher UX: single launcher vs one command per feature; docked pane remains a later hosting variant | Single launcher window first; verify U11 separately | Step 03 |
| D2 | Parameter editing: file-only vs in-UI | File-only + read-only display | Step 02 |
| D3 | Highlight mechanism | `IOpenGLService` overlay if verified non-persistent | Step 05 |
| D4 | Excel library (or CSV instead) | Evaluate candidates; approve one | Step 06 |
| D5 | Probe features in client package (hidden) or dev-only | Dev-only unless needed for on-site support | Step 04 |
| D6 | Automation success semantics: map successful Automation execution onto the existing AGENTS §10 states, or formally add `Succeeded` for Automations only | Decide before finalizing the Step 01 result-state enum. Any AGENTS.md change requires explicit user approval. | Step 01 |

## 9. Inputs needed from the customer (do not block Steps 01–05)

- Sample mold assembly in Cimatron format with representative attributes and color codes (for `ModelInspectionProbe`).
- Sample DO Input Sheet and slider weight/diameter table (for Step 06 column mapping).
- Attribute names/values planned for steel grade, springs, ledges, supports and O-rings (CR-04, 08, 26).
- Customer color-code standard (part area, vents, cooling IN/OUT).

---

## 10. Definition of done for the platform (after Step 06)

- A new CR can be added without platform changes when it needs only existing ports. The work is:
  - one feature class in DesignChecks;
  - pure rules in Domain;
  - one registration in `FeatureComposition`;
  - its parameters in config;
  - its tests.
- All steps pass the AGENTS §17 quality gates, with user-supplied evidence.
- `CR-PLATFORM-GUIDE.md` exists and matches the implemented platform.
- All U-items used by the platform are verified and documented with source and Cimatron version, or explicitly return `Unsupported`.
