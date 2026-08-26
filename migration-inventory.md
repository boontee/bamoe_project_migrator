# BAMOE v8 → v9.5 Migration Inventory

**Project:** `evaluation` (`evaluation:evaluation:1.0.0-SNAPSHOT`)
**Source directory (v8):** `v8-projects/v8-app/`
**Output directory (v9):** `v9-projects/v9-app/`
**Audit date:** 2025-01-01
**Audited by:** BAMOE Migrator

---

## 1. Project Identity

| Property | Value |
|---|---|
| Maven coordinates | `evaluation:evaluation:1.0.0-SNAPSHOT` |
| Packaging | `kjar` |
| Display name | `Evaluation_Process` |
| Description | Getting started Business Process for evaluating employees |
| v8 KIE version | `7.67.2.Final-redhat-00017` |
| Build Java runtime | OpenJDK 11.0.3 (from `build.metadata`) |
| Build Maven version | 3.6.3 (from `build.metadata`) |

---

## 2. Asset Inventory

### 2.1 BPMN Workflow Files

| File | Process ID | Process Name | Issues |
|---|---|---|---|
| `src/main/resources/evaluation.bpmn` | **`evaluation`** | Evaluation | See §3.1 |

**Process structure summary:**
- 1 Start Event
- 3 User Tasks: `Self Evaluation`, `PM Evaluation`, `HR Evaluation`
- 2 Parallel Gateways (diverging + converging)
- 1 Terminate End Event
- No Script Tasks
- No intermediate events
- No subprocesses
- No call activities
- No service tasks / custom WIHs

### 2.2 DMN Decision Files

| File | Notes |
|---|---|
| _(none found)_ | No `.dmn` files present |

### 2.3 DRL Rules / Guided Rules / DSL

| File | Notes |
|---|---|
| _(none found)_ | No `.drl`, `.gdrl`, `.dslr`, `.dsl` files present |

### 2.4 Decision Tables (XLS / XLSX / Guided)

| File | Notes |
|---|---|
| _(none found)_ | No `.xlsx`, `.xls`, `.gdst` files present |

### 2.5 Test Scenarios

| File | Notes |
|---|---|
| _(none found)_ | No `.scesim` files present |

### 2.6 Forms (`.frm`)

| File | Type | Fields | Status |
|---|---|---|---|
| `src/main/resources/evaluation-taskform.frm` | Process start form | `employee` (TextBox), `reason` (TextArea) | **Must replace** — v8 `.frm` format is incompatible with v9 |
| `src/main/resources/PerformanceEvaluation-taskform.frm` | User Task form | `reason` (read-only TextArea), `performance` (IntegerBox) | **Must replace** — v8 `.frm` format is incompatible with v9 |

### 2.7 Work Item Definitions (`.wid`)

| File | Notes |
|---|---|
| _(none found)_ | No `.wid` files present |

### 2.8 Custom WorkItemHandlers (Java)

| Class | Notes |
|---|---|
| _(none found)_ | `src/main/java/` is empty; no WIH classes present |

### 2.9 Custom Security / Identity Classes

| Class | Notes |
|---|---|
| _(none found)_ | No `UserGroupCallback`, `AssignmentStrategy`, or `TaskLifeCycleEventListener` implementations present |

### 2.10 Custom Event Listeners

| Class | Notes |
|---|---|
| _(none found)_ | `kie-deployment-descriptor.xml` shows empty `<event-listeners/>` and `<task-event-listeners/>` |

### 2.11 SVG Diagrams

| File | Notes |
|---|---|
| `src/main/resources/evaluation-svg.svg` | Migrate as-is; Kogito serves SVGs automatically |

### 2.12 PMML Models

| File | Notes |
|---|---|
| _(none found)_ | No PMML files present |

### 2.13 Case Management (CMMN)

| File | Notes |
|---|---|
| _(none found)_ | No CMMN files present; project is pure BPMN |

### 2.14 v8-Specific Infrastructure Files

| File | v9 Status |
|---|---|
| `src/main/resources/META-INF/kmodule.xml` | **Must delete** — not used in v9 |
| `src/main/resources/META-INF/kie-deployment-descriptor.xml` | **Must delete** — not used in v9 |
| `src/main/resources/META-INF/persistence.xml` | **Must delete** — Quarkus manages persistence via `application.properties` |
| `project.imports` | **Must delete** — Business Central artefact, not used in v9 |
| `project.repositories` | **Must delete** — Business Central artefact, not used in v9 |
| `build.metadata` | **Must delete** — build-time metadata, not used in v9 |

---

## 3. Blockers and Required Changes

### 3.1 BPMN: Process ID Check

| Process ID | Contains hyphen? | Contains dot? | Action required |
|---|---|---|---|
| `evaluation` | No | No | ✅ Safe — no change needed |

### 3.2 BPMN: Script Task Language Check

No Script Tasks are present in the process. ✅ Not applicable.

### 3.3 BPMN: Gateway Expression Language Check

Both parallel gateways are unconditional (no conditions on outgoing flows). No DROOLS-dialect expressions are used. ✅ Not applicable.

### 3.4 BPMN: expressionLanguage Attribute

The `<bpmn2:definitions>` element declares `expressionLanguage="http://www.mvel.org/2.0"`. In v9, MVEL is not used for gateway conditions; this attribute is harmless at the definition level since there are no conditional sequence flows. ✅ No action required.

### 3.5 BPMN: Shared KIE-Session Dependencies (insert/retract)

No `insert()` or `retract()` calls appear in the BPMN or any rules file. ✅ Not applicable.

### 3.6 Forms: `.frm` Incompatibility

Both `.frm` files use the v8 `org.kie.workbench.common.forms.*` schema.
This format is **completely incompatible** with v9. New forms must be generated using the **BAMOE Developer Tools for VS Code** "Generate form code for User Task" command.

| Form | Bound to | Fields to reproduce |
|---|---|---|
| `evaluation-taskform.frm` | Process `evaluation` start | `employee` (String, required), `reason` (String, required) |
| `PerformanceEvaluation-taskform.frm` | User Task `PerformanceEvaluation` | `reason` (String, read-only), `performance` (Integer, required) |

### 3.7 pom.xml: KJAR Packaging and v8 Dependencies

| Issue | Detail | Action |
|---|---|---|
| `<packaging>kjar</packaging>` | Not valid in v9 | Replace with standard `jar` or `quarkus-app` |
| `kie-maven-plugin` 7.67 | Replaced by Quarkus/Kogito build | Remove plugin; add `kogito-quarkus-bom` or `bamoe-bom` |
| `kie-api` / `kie-internal` 7.67 | v8 KIE API | Remove; replaced by Kogito APIs |
| `junit:junit:4.12` | EOL; not supported in v9 test harness | Replace with `quarkus-junit5` |
| `xstream:1.4.11` | CVE-affected old version; v8-only | Remove |
| Missing `groupId` for parent BOM | No parent POM / BOM declared | Add BAMOE 9.5 BOM as `dependencyManagement` import |

### 3.8 application.properties: v8 Property Keys

No `application.properties` exists in the v8 project (it was a KJAR, not a runnable app). Properties must be created from scratch in the v9 project with `kogito.*` keys.

### 3.9 javax.* Imports

No Java source files exist in `src/main/java/`. ✅ Not applicable.

### 3.10 TaskName Assignment on User Tasks

All three User Tasks assign `TaskName = "PerformanceEvaluation"`. In v9, the task name is derived from the BPMN task `name` attribute automatically, but the `TaskName` data input is still supported. Flag for verification during Sub-Task 3.

---

## 4. Asset Classification Summary

| Asset | Category | Sub-Task |
|---|---|---|
| `evaluation.bpmn` | **Requires manual update** (process ID safe; expressionLanguage attribute; TaskName check) | Sub-Task 3 |
| `evaluation-taskform.frm` | **Must replace** (v8 `.frm` incompatible) | Sub-Task 8 |
| `PerformanceEvaluation-taskform.frm` | **Must replace** (v8 `.frm` incompatible) | Sub-Task 8 |
| `evaluation-svg.svg` | **Migrate as-is** | Sub-Task 3 |
| `META-INF/kmodule.xml` | **Must delete** | Sub-Task 2 |
| `META-INF/kie-deployment-descriptor.xml` | **Must delete** | Sub-Task 2 |
| `META-INF/persistence.xml` | **Must delete** | Sub-Task 2 |
| `project.imports` | **Must delete** | Sub-Task 2 |
| `project.repositories` | **Must delete** | Sub-Task 2 |
| `build.metadata` | **Must delete** | Sub-Task 2 |
| `pom.xml` | **Must replace** (kjar packaging, v8 deps) | Sub-Task 2 |

---

## 5. Features NOT Present (no action required)

The following items from the migration checklist are **not present** in this project and require no migration work:

- DMN files
- DRL / guided rules / guided decision tables / DSL / DSLR / scorecards
- PMML models
- Case Management (CMMN)
- Test scenarios (`.scesim`)
- Work Item Definitions (`.wid`)
- Custom WorkItemHandlers
- Custom `UserGroupCallback` / `AssignmentStrategy`
- Custom `TaskLifeCycleEventListener`
- Custom event listeners
- Script Tasks (JavaScript or MVEL)
- DROOLS gateway expressions
- Shared KIE-session dependencies
- `javax.*` imports in domain classes

---

## 6. Environment Delta

| Dimension | v8 (current) | v9.5 (target) |
|---|---|---|
| Java | OpenJDK 11 | **Java 21** |
| Maven | 3.6.3 | **Maven 3.9.11+** |
| Runtime framework | KJAR on KIE Server / JBoss EAP | **Quarkus 3.x (Kogito)** |
| Packaging | `kjar` | `quarkus-app` (uber-jar or native) |
| BOM | `kie-maven-plugin 7.67.2.Final-redhat-00017` | **BAMOE 9.5 BOM** |
| Persistence | JPA / JTA via `persistence.xml` (`org.jbpm.domain`) | Kogito managed via `application.properties` |
| Process metadata | `kmodule.xml` + `kie-deployment-descriptor.xml` | `pom.xml` + `application.properties` only |

---

## 7. Risk Register

| ID | Risk | Severity | Notes |
|---|---|---|---|
| R1 | Forms must be fully regenerated | Medium | Field names and types are documented above; logic is straightforward |
| R2 | User Task name `PerformanceEvaluation` used as `TaskName` input on all 3 tasks (Self, PM, HR) | Low | All three tasks share the same task form in v8; verify correct form binding per task in v9 |
| R3 | `expressionLanguage="http://www.mvel.org/2.0"` on `<bpmn2:definitions>` | Low | No conditional flows use MVEL; attribute may be stripped or ignored by Kogito |
| R4 | JUnit 4.12 and XStream 1.4.11 in test scope | Low | No test classes exist; these deps will simply be removed |
| R5 | `persistence.xml` references `java:jboss/datasources/ExampleDS` (JBoss-specific JNDI) | Low | Not carried forward; v9 uses standard Quarkus datasource properties |
