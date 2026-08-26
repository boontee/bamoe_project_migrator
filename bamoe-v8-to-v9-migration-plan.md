# BAMOE v8 to v9.5 Migration Plan

## Top-Level Overview

This plan covers the full migration of an IBM Business Automation Manager Open Editions (BAMOE) v8.0.x
project to BAMOE v9.5 (latest). The migration represents a **fundamental architectural shift** — not merely
a version bump — from a centralized, monolithic Business Central / KIE Server model to a cloud-native,
microservice-based model built on Kogito + Quarkus (or Spring Boot).

**Sources consulted:**
- `https://github.com/IBM/bamoe-docs/blob/9.5.x/nav.adoc` (9.5.x doc tree)
- All upgrade sub-sections under `https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/`

**Scope:** Project structure, POM/dependencies, application configuration, all asset types (BPMN,
DMN, DRL, decision tables, forms, test scenarios), security/auth, event streaming, runtime deployment,
and live-data strategy.

**Non-goals:** This plan does not cover infrastructure provisioning (Kubernetes/OpenShift cluster setup)
or CI/CD pipeline creation beyond what is required to validate the migrated application.

**Directory convention:**
- v8 source projects live under `v8-projects/<project-name>/` (e.g. `v8-projects/v8-app/`).
- The corresponding v9 output is placed under `v9-projects/<v9-name>/` where `<v9-name>` is the
  original folder name with the `v8-` prefix replaced by `v9-` (e.g. `v8-app` → `v9-app`).
- All sub-tasks that read from or write to the project must use these explicit paths.

---

## Prerequisites — Developer Environment Setup

> **Must be satisfied before any sub-task begins. These are not implementation steps — they are
> gate conditions. Do not proceed with Sub-Task 1 until every item below is checked off.**

### Required toolchain

| Tool | Minimum version | Verify with |
|---|---|---|
| Java (IBM Semeru or Eclipse Temurin) | **21** | `java -version` |
| Maven | **3.9.11** | `mvn -version` |
| Git | any recent | `git --version` |
| VS Code | any recent | — |
| BAMOE Developer Tools VS Code extension | 9.5.x | VS Code Extensions panel |

### Required BAMOE artefacts

- **BAMOE Maven Repository** — pull the container image
  `icr.io/cpopen/bamoe/bamoe-maven-repository-volume` (or install locally into `~/.m2`) and
  configure `~/.m2/settings.xml` to resolve from it.
  See `installation/development-environment.adoc#maven-repo-setup`.
- **BAMOE Canvas** _(optional, for business-analyst authoring)_ — deploy via Helm chart following
  `installation/development-environment.adoc#canvas-installation`.

### Verification gate

Run the following against a minimal `pom.xml` that imports the Kogito BOM:

```bash
mvn dependency:resolve
```

All dependencies must resolve without error before proceeding.

### References
- `installation/development-environment.adoc`
- `https://github.com/IBM/bamoe-canvas-quarkus-accelerator` (reference `pom.xml` per target branch)
- `release-notes/supported-environments.adoc`

---

## Sub-Task 1 — Audit the v8 Project and Classify Assets

**Status:** `[x] done`

### Intent
Before touching any code, produce a complete inventory of everything in the v8 project that must be
migrated, converted, or dropped. This surfaces blockers early and sizes the remaining sub-tasks.

### Expected Outcomes
- A written inventory document listing every asset type present (BPMN, DMN, DRL, XLSX/XLS decision tables,
  PMML, guided rules/decision tables, DSL/DSLR, forms `.frm`, test scenarios `.scesim`, WIDs, WIHs,
  custom event listeners, custom UserGroupCallback/AssignmentStrategy implementations).
- Clear categorisation of each asset as: **Migrate as-is**, **Requires manual update**, or **Must replace**.
- Identification of JavaScript scripting in Script Tasks (not supported in v9).
- Identification of PMML models (not supported in v9).
- Identification of Case Management (CMMN) projects.
- Identification of shared KIE-session dependencies between BPMN and DRL (requires refactor).
- List of process IDs containing hyphens or dots (break code generation in v9).
- List of raw `List` types and `javax.*` imports in domain classes.

### Todo List
1. Run `find . -type f` on `v8-projects/v8-app/`; group files by extension and directory.
2. For each BPMN file: note process IDs, Script Task languages, gateway expression languages
   (DROOLS vs Java), and any shared-KIE-session pattern (`insert()`, `retract()` called from BPMN).
3. For each DRL file: flag rules using `insert()`/`retract()` in a `ruleflow-group` invoked from BPMN.
4. List all `.wid` files and corresponding WIH Java classes.
5. List all custom `UserGroupCallback`, `AssignmentStrategy`, and `TaskLifeCycleEventListener` classes.
6. List all `.frm` form files (must be recreated from scratch in v9).
7. Check for PMML, guided rules, guided decision tables, DSL/DSLR, and scorecards.
8. Check for Case Management (CMMN) assets.
9. Record the current Java version, Maven version, Quarkus/Spring Boot version.
10. Record current `application.properties` keys (all `kieserver.*`, `drools.*`, `jbpm.*` properties
    will be replaced with `kogito.*`).
11. Produce `migration-inventory.md` at the workspace root summarising findings.
    - Record the v8 source path (`v8-projects/v8-app/`) and the v9 output path (`v9-projects/v9-app/`).

### Knowledge Sources
> Read these documents before running the audit. Every item in the inventory checklist maps to at
> least one of the sources below.

| What to check | Authoritative source |
|---|---|
| Full list of unsupported features to flag | [`upgrade/11-not-supported.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) |
| Architectural changes (what changed and why) | [`upgrade/02-key-differences.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/02-key-differences.adoc) |
| Project file differences (files to remove vs keep) | [`upgrade/05-01-upgrading-structure-differences.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-01-upgrading-structure-differences.adoc) |
| Sample projects and asset matrix | [`upgrade/12-sample-project-references.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/12-sample-project-references.adoc) |
| Frequently asked migration questions | [`upgrade/13-faq.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/13-faq.adoc) |

---

## Sub-Task 2 — Migrate Project Structure and POM

**Status:** `[x] done`

### Intent
Replace the v8 KJAR/Business-Central project structure with a valid v9 Maven project that compiles
cleanly before any assets are migrated.

### Expected Outcomes
- `pom.xml` replaced: v8 RHPAM BOM removed; Kogito + Quarkus (or Spring Boot) BOM and extensions added.
- Removed files: `kmodule.xml`, `kie-deployment-descriptor.xml`, `persistence.xml` (moved to
  `application.properties`), `project.repositories`, `project.imports`, `package-names-white-list`.
- `application.properties` created with v9 skeleton (`kogito.*`, `quarkus.*` or `spring.*` keys).
- `mvn clean package` succeeds (even if assets are not yet migrated — they can be temporarily excluded).

### Todo List
1. Create the v9 output directory at `v9-projects/v9-app/` (mirrors `v8-projects/v8-app/` with the
   `v8-` prefix replaced by `v9-`).
2. Download the `pom.xml` from the branch of `https://github.com/IBM/bamoe-canvas-quarkus-accelerator`
   matching the target v9.5 release as the base template; write it to `v9-projects/v9-app/pom.xml`.
3. Merge project-specific `groupId`, `artifactId`, `version`, and any non-BAMOE dependencies from
   `v8-projects/v8-app/pom.xml` into the new template.
4. Replace RHPAM/KIE BOM entries with the Kogito + Quarkus BOM equivalents from the BAMOE Maven repo;
   see `reference-guide/maven-repository-libraries.adoc`.
5. Do **not** copy `kmodule.xml`, `kie-deployment-descriptor.xml`, `persistence.xml`,
   `project.repositories`, `project.imports`, or `package-names-white-list` into the v9 project.
6. Create `v9-projects/v9-app/src/main/resources/application.properties` with the v9 skeleton
   (HTTP port, log level, datasource if persistence is needed, `kogito.persistence.type=jdbc` if applicable).
7. Add `org.drools:drools-decisiontables` dependency if XLS/XLSX decision tables are present (found
   in Sub-Task 1 inventory).
8. Add `org.kie.kogito:kogito-scenario-simulation` if `.scesim` test scenarios are present.
9. Run `mvn clean package -DskipTests` from `v9-projects/v9-app/`; resolve any compilation errors.

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. When in doubt about a configuration value or dependency, consult the source listed
> next to that step — do not infer from v8 knowledge.

| Step | Authoritative source |
|---|---|
| Replace `pom.xml` / BOM entries | [`upgrade/05-02-upgrading-client-server-projects.adoc` — The POM file](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-02-upgrading-client-server-projects.adoc) |
| Project structure comparison | [`upgrade/05-01-upgrading-structure-differences.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-01-upgrading-structure-differences.adoc) |
| `application.properties` skeleton (Quarkus & Spring Boot) | [`upgrade/05-02-upgrading-client-server-projects.adoc` — application properties](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-02-upgrading-client-server-projects.adoc) |
| Selecting correct Maven dependencies | [`reference-guide/maven-repository-libraries.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/reference-guide/maven-repository-libraries.adoc) |
| Files to delete (`kmodule.xml`, `kie-deployment-descriptor.xml`, etc.) | [`upgrade/05-01-upgrading-structure-differences.adoc` — Project files comparison table](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-01-upgrading-structure-differences.adoc) |
| Accelerator reference `pom.xml` | [github.com/IBM/bamoe-canvas-quarkus-accelerator](https://github.com/IBM/bamoe-canvas-quarkus-accelerator) — use branch matching v9.5 |

---

## Sub-Task 3 — Migrate BPMN Workflow Assets

**Status:** `[ ] pending`

### Intent
Open every BPMN model in the v9 tooling to trigger the non-destructive XML upgrade, then address
all v9-incompatible constructs identified in Sub-Task 1.

### Expected Outcomes
- All BPMN files saved through BAMOE Developer Tools or Canvas (XML IDs updated, character escaping
  applied automatically).
- No Script Tasks using JavaScript (replaced with Java).
- No gateway conditions using DROOLS expressions (replaced with Java expressions).
- No `insert()` / `retract()` / `update()` / `delete()` called from BPMN Business Rules tasks via
  shared KIE session; data is now passed via Input/Output mappings.
- Process IDs contain no hyphens or dots.
- BPMN variable types use fully qualified class names (no auto-complete in v9 editor).
- Case Management (CMMN) assets converted to Flexible (AdHoc) BPMN Processes.
- WID files verified to be present in project root or alongside BPMN files.

### Todo List
1. Open each BPMN file in BAMOE Developer Tools; make a trivial change and save to trigger XML upgrade.
2. For every Script Task with `dialect="javascript"`: rewrite the script in Java.
3. For every gateway using a DROOLS expression: replace with an equivalent Java expression.
4. For every Business Rules task calling a `ruleflow-group` that relies on `insert()`/`retract()`:
   a. Refactor the DRL rule to remove `insert()`/`retract()` and operate on mapped data directly.
   b. Add Input/Output data mappings on the Business Rules task to pass data explicitly.
   c. Remove any "Retract" DRL tasks that exist solely to clean up shared working memory.
5. Rename any process IDs containing hyphens (`-`) to camelCase; update all references.
6. For each BPMN variable referencing a Java type: manually enter the fully qualified class name.
7. For Case Management projects: follow `upgrade/05-10-upgrading-case-management-project.adoc`
   to convert to AdHoc/Flexible Processes (set `AdHoc=true`).
8. Verify `.wid` files are present and not using the deprecated `@Wid` annotation (see Sub-Task 5).
9. Run `mvn clean package`; resolve any BPMN-related code generation errors.

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. Consult the linked source before modifying any BPMN construct.

| Step | Authoritative source |
|---|---|
| Open/save BPMN to trigger XML upgrade | [`upgrade/05-04-upgrading-individual-assets.adoc` — Workflows (BPMN)](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| Replace JavaScript scripting with Java | [`upgrade/11-not-supported.adoc` — JavaScript Scripting row](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) |
| Replace DROOLS gateway expressions with Java | [`upgrade/05-04-upgrading-individual-assets.adoc` — BPMN and DRLs section](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| Remove `insert()`/`retract()` shared KIE-session patterns | [`upgrade/05-04-upgrading-individual-assets.adoc` — BPMN and DRLs example](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| Rename process IDs (no hyphens / dots) | [`upgrade/11-not-supported.adoc` — Process ID restrictions row](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) |
| BPMN variable fully-qualified class names | [`upgrade/05-04-upgrading-individual-assets.adoc` — BPMN and Java classes](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| Case Management → Flexible (AdHoc) Processes | [`upgrade/05-10-upgrading-case-management-project.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-10-upgrading-case-management-project.adoc) |
| WID file placement and `@Wid` removal | [`upgrade/05-04-upgrading-individual-assets.adoc` — Work Item Definitions (WIDs)](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) and [`upgrade/05-05-upgrading-custom-work-item-handlers.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-05-upgrading-custom-work-item-handlers.adoc) |
| Business Calendar migration (`calendar.properties`) | [`upgrade/05-07-upgrading-business-calendar.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-07-upgrading-business-calendar.adoc) |

---

## Sub-Task 4 — Migrate DMN Decision Assets

**Status:** `[ ] pending`

### Intent
Upgrade DMN models to DMN 1.6 via the new editor, ensuring PMML dependencies are removed and
Java-backed types are recreated using Developer Tools.

### Expected Outcomes
- All DMN files upgraded to DMN 1.6 and saved through Developer Tools or Canvas.
- Any PMML models removed; replacement strategy documented (external ML integration or DMN decision tables).
- Java-backed DMN types recreated using the Developer Tools (`Import DMN data types from Java classes` feature).
- DMN test scenarios (`.scesim`) open and pass in Developer Tools.

### Todo List
1. Open each DMN file in BAMOE Developer Tools; make a trivial change and save (triggers upgrade to DMN 1.6).
2. For any DMN model referencing a PMML model: remove the reference; document replacement approach
   (use DMN decision tables or integrate an external ML service).
3. For Java-backed DMN types: use Developer Tools `Import DMN data types from Java classes`
   to recreate them; verify the embedded type is usable in Canvas too.
4. Open each DMN-based `.scesim` in Developer Tools; save without changes; run `mvn test` to confirm.
5. Run `mvn clean package`; resolve any DMN-related compilation errors.

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. Consult the linked source before modifying any DMN model.

| Step | Authoritative source |
|---|---|
| Open/save DMN to upgrade to DMN 1.6 | [`upgrade/05-04-upgrading-individual-assets.adoc` — Decisions (DMN)](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| Remove PMML references; choose replacement | [`upgrade/11-not-supported.adoc` — PMML row](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) and [`upgrade/05-04-upgrading-individual-assets.adoc` — Predictive models (PMML)](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| Java-backed DMN types via Developer Tools | [`tools/importing-dmn-data-types-from-java-classes.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/tools/importing-dmn-data-types-from-java-classes.adoc) |
| DMN test scenarios (`.scesim`) tooling support | [`upgrade/05-04-upgrading-individual-assets.adoc` — Test Scenarios (SCESIM)](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |

---

## Sub-Task 5 — Migrate DRL Rules, Decision Tables, and Test Scenarios

**Status:** `[ ] pending`

### Intent
DRL rules are largely compatible in v9, but guided rules, DSL/DSLR, scorecards, and PMML must be
converted. Test scenario activators must be updated. Decision tables need a dependency addition only.

### Expected Outcomes
- All guided rules, guided decision tables, DSL/DSLR, and scorecards converted to DRL or DMN.
- XLS/XLSX decision tables retained; `org.drools:drools-decisiontables` dependency confirmed in POM.
- DRL files reviewed: no remaining shared KIE-session patterns with BPMN (addressed in Sub-Task 3).
- JUnit activator in `.scesim` test files updated from `@RunWith(ScenarioJunitActivator.class)` to
  `@TestScenarioActivator` (JUnit 5).
- All test scenarios execute cleanly via `mvn test`.

### Todo List
1. Convert each guided rule to DRL syntax (see `upgrade/11-not-supported.adoc` example).
2. Convert each guided decision table to DRL or XLS/XLSX decision table format.
3. Convert DSL/DSLR files to DRL.
4. Convert scorecard assets to DMN decision tables.
5. In each `.scesim` activator Java class: replace `@RunWith(org.drools.scenariosimulation.backend.runner.ScenarioJunitActivator.class)` with `@org.drools.scenariosimulation.backend.runner.TestScenarioActivator`.
6. Confirm `org.kie.kogito:kogito-scenario-simulation` is in the POM (added in Sub-Task 2).
7. Note: DRL-based and DMN-based test scenarios cannot coexist in the same project in v9 — split into
   separate modules if both are present.
8. Run `mvn test`; resolve failures.

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. Consult the linked source before converting any rule or test artifact.

| Step | Authoritative source |
|---|---|
| Convert Guided Rules to DRL | [`upgrade/11-not-supported.adoc` — Guided Rules row (with DRL example)](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) |
| Convert Guided Decision Tables | [`upgrade/11-not-supported.adoc` — Guided Decision Tables row](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) |
| Convert DSL / DSLR to DRL | [`upgrade/02-key-differences.adoc` — DSL and DSLR section](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/02-key-differences.adoc) |
| Convert Scorecards to DMN | [`upgrade/11-not-supported.adoc` — Scorecards row](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) |
| DRL compatibility and Rule Units option | [`upgrade/05-04-upgrading-individual-assets.adoc` — Rules (DRL)](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| XLS/XLSX decision tables — required dependency | [`upgrade/05-04-upgrading-individual-assets.adoc` — Decision Tables](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| JUnit activator migration (`@TestScenarioActivator`) | [`upgrade/05-04-upgrading-individual-assets.adoc` — JUnit activator migration](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |
| DRL-based vs DMN-based test scenario coexistence limit | [`upgrade/05-04-upgrading-individual-assets.adoc` — Tooling support](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-04-upgrading-individual-assets.adoc) |

---

## Sub-Task 6 — Migrate Custom WorkItemHandlers

**Status:** `[ ] pending`

### Intent
Replace the v8 `WorkItemHandler` API with the v9 Kogito phase-based API, update registration from
XML descriptors to a programmatic `DefaultWorkItemHandlerConfig` bean, and remove `@Wid` annotations.

### Expected Outcomes
- Every custom WIH class extends `org.kie.kogito.process.workitems.impl.DefaultKogitoWorkItemHandler`
  and overrides `activateWorkItemHandler()` / `abortWorkItemHandler()` instead of `executeWorkItem()` / `abortWorkItem()`.
- WIH registration moved from `kie-deployment-descriptor.xml` to a `@ApplicationScoped` bean
  extending `org.kie.kogito.process.impl.DefaultWorkItemHandlerConfig`.
- `@Wid` annotations removed from WIH classes; `.wid` descriptor files retained in project root.
- Retry logic reconfigured via `kogito.faultToleranceEnabled` and Jobs Service properties
  instead of the v8 Executor mechanism.
- All WIH-related unit tests pass.

### Todo List
1. For each WIH class: change `implements WorkItemHandler` to `extends DefaultKogitoWorkItemHandler`.
2. Rename `executeWorkItem(WorkItem, WorkItemManager)` to `activateWorkItemHandler(...)` with the
   v9 signature; adapt internal logic accordingly.
3. Rename `abortWorkItem(WorkItem, WorkItemManager)` to `abortWorkItemHandler(...)`.
4. Remove any `@Wid` annotations from WIH classes.
5. Create one `@ApplicationScoped` class extending `DefaultWorkItemHandlerConfig`; register all
   custom handlers with `register("HandlerName", new MyHandler())`.
6. Configure retry logic in `application.properties`:
   - `kogito.faultToleranceEnabled=true`
   - `kogito.jobs-service.maxNumberOfRetries=3`
   - `kogito.jobs-service.retryMillis=60000`
7. Run `mvn clean package` and unit tests; resolve errors.

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. Consult the linked source before modifying any WorkItemHandler class.

| Step | Authoritative source |
|---|---|
| API change: `WorkItemHandler` → `DefaultKogitoWorkItemHandler` | [`upgrade/05-05-upgrading-custom-work-item-handlers.adoc` — API and lifecycle changes](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-05-upgrading-custom-work-item-handlers.adoc) |
| Registration change: XML → `DefaultWorkItemHandlerConfig` bean | [`upgrade/05-05-upgrading-custom-work-item-handlers.adoc` — Registration changes](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-05-upgrading-custom-work-item-handlers.adoc) |
| `@Wid` annotation removal; `.wid` file placement | [`upgrade/05-05-upgrading-custom-work-item-handlers.adoc` — Annotation support](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-05-upgrading-custom-work-item-handlers.adoc) |
| Retry logic: Jobs Service vs v8 Executor | [`upgrade/05-05-upgrading-custom-work-item-handlers.adoc` — Retry mechanisms](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-05-upgrading-custom-work-item-handlers.adoc) |
| Predefined WIH availability (dropped handlers list) | [`upgrade/11-not-supported.adoc` — Predefined WorkItemHandlers row](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) |
| Tutorial: database integration WIH end-to-end | Tutorial 12 in the BAMOE v8→v9 migration tutorials repo (linked from `upgrade/05-05-upgrading-custom-work-item-handlers.adoc`) |

---

## Sub-Task 7 — Migrate Security, Auth, UserGroupCallback, and AssignmentStrategy

**Status:** `[ ] pending`

### Intent
Replace WildFly/JAAS-based authentication with OIDC (Keycloak or any OIDC-compliant IdP), migrate
`UserGroupCallback` to `IdentityProvider`, and replace `AssignmentStrategy` with `UserTaskAssignmentStrategy`.

### Expected Outcomes
- No `UserGroupCallback` implementations remain; user/group resolution delegated to OIDC identity provider.
- `IdentityProvider` interface used for identity resolution in both secured and non-secured modes.
- OIDC configured in `application.properties` for Quarkus (`quarkus.oidc.*`) or Spring Boot
  (`spring.security.oauth2.*`).
- Custom `AssignmentStrategy` replaced with `UserTaskAssignmentStrategy.computeAssignment()`.
- BAMOE Management Console OIDC proxy dependencies added to POM if Management Console is used.
- Direct LDAP integration replaced by IdP group membership via OIDC token claims.

### Todo List
1. Remove all `UserGroupCallback` implementations (JAASUserGroupCallbackImpl, DBUserGroupCallbackImpl,
   LDAPUserGroupCallbackImpl, PropsUserGroupCallbackImpl, or custom).
2. Implement `IdentityProvider` or rely on `QuarkusIdentityProvider` (injected automatically in Quarkus).
3. Configure OIDC in `application.properties`:
   - For Quarkus: add `quarkus-oidc` dependency; set `quarkus.oidc.auth-server-url`, `client-id`, `credentials.secret`.
   - For Spring Boot: add `spring-boot-starter-oauth2-resource-server`; configure `spring.security.oauth2.*`.
4. For non-secured dev/test mode: set `kogito.security.auth.enabled=false` and use query params
   (`?user=admin&group=Manager`) for task endpoints.
5. Migrate custom `AssignmentStrategy` to implement `UserTaskAssignmentStrategy`; override
   `computeAssignment(UserTaskInstance, IdentityProvider)`.
6. If Management Console is used: add `quarkus-oidc-proxy` and
   `quarkus-resteasy-client-oidc-token-propagation` dependencies; configure OIDC tenant in `application.properties`.
7. Run integration test (start service, request a token from IdP, call a secured endpoint).

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. Consult the linked source before modifying any security, identity, or auth class.

| Step | Authoritative source |
|---|---|
| OIDC setup (Quarkus `quarkus-oidc`; Spring Boot OAuth2) | [`upgrade/05-08-upgrading-authentication-authorization.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-08-upgrading-authentication-authorization.adoc) |
| `UserGroupCallback` → `IdentityProvider` migration | [`upgrade/05-06-upgrading-user-groups-callback.adoc` — Upgrading UserGroupCallback](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-06-upgrading-user-groups-callback.adoc) |
| Non-secured mode (`kogito.security.auth.enabled=false`) | [`upgrade/05-06-upgrading-user-groups-callback.adoc` — Behaviour in non-secured applications](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-06-upgrading-user-groups-callback.adoc) |
| `AssignmentStrategy` → `UserTaskAssignmentStrategy` | [`upgrade/05-06-upgrading-user-groups-callback.adoc` — Upgrading AssignmentStrategy](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-06-upgrading-user-groups-callback.adoc) |
| Management Console OIDC proxy wiring | [`upgrade/05-08-upgrading-authentication-authorization.adoc` — BAMOE Management Console additional configuration](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-08-upgrading-authentication-authorization.adoc) |
| Securing REST endpoints end-to-end | [`building-deploying/security.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/building-deploying/security.adoc) |
| Tutorial: security migration end-to-end | Tutorial 11 in the BAMOE v8→v9 migration tutorials repo (linked from `upgrade/05-06-upgrading-user-groups-callback.adoc`) |

---

## Sub-Task 8 — Migrate Forms

**Status:** `[ ] pending`

### Intent
v8 `.frm` form files are completely incompatible with v9. All forms must be regenerated using the
Developer Tools form-code-generation command.

### Expected Outcomes
- All v8 `.frm` files deleted or archived.
- New form code generated for every User Task using Developer Tools
  (`Generate form for selected User Task` command).
- Forms reviewed for correctness and adapted as needed for the target UI.

### Todo List
1. Archive (do not delete yet) all `.frm` and `patterns.json` / `themes.json` files.
2. For each User Task in the migrated BPMN models: use Developer Tools to run
   `Generate form code for User Task` and save the output.
3. Validate generated forms by starting the service in Quarkus Dev Mode and exercising each User Task.
4. Customise generated form code to match v8 UX/business requirements.
5. Delete archived `.frm` files once form functionality is verified.

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. Consult the linked source before regenerating any form.

| Step | Authoritative source |
|---|---|
| Why v8 `.frm` files cannot be reused | [`upgrade/11-not-supported.adoc` — Legacy Form Modeler row](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/11-not-supported.adoc) and [`upgrade/02-key-differences.adoc` — Forms section](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/02-key-differences.adoc) |
| Generate form code using Developer Tools | [`tools/form-generation.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/tools/form-generation.adoc) |
| `patterns.json` / `themes.json` not carried forward | [`upgrade/05-01-upgrading-structure-differences.adoc` — Project files comparison table](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-01-upgrading-structure-differences.adoc) |

---

## Sub-Task 9 — Migrate Event Streaming and Event Listeners

**Status:** `[ ] pending`

### Intent
Replace v8 KIE Server Kafka event emission and `TaskLifeCycleEventListener` implementations
with v9 CloudEvents-based add-ons and the `UserTaskEventListener` / `ProcessEventListener` interfaces.

### Expected Outcomes
- Process and task events emitted in CloudEvents format via the process-event add-on.
- Messaging events (BPMN message elements) wired to Kafka via the messaging add-on.
- All `TaskLifeCycleEventListener` implementations replaced with `UserTaskEventListener`
  (single-callback, old+new state pattern).
- Kafka configuration using `mp.messaging.*` (Quarkus) or `spring.kafka.*` (Spring Boot) properties.
- CDI `@ApplicationScoped` annotation used for listener registration (no manual registration needed).

### Todo List
1. Add the Kogito process-event and/or messaging add-on dependencies to `pom.xml` (see Maven repo libs).
2. For each `TaskLifeCycleEventListener` class:
   a. Change to `implements UserTaskEventListener`.
   b. Merge `beforeTaskXXX()` + `afterTaskXXX()` into a single `onUserTaskState(UserTaskStateEvent)` method,
      using `event.getOldStatus()` and `event.getNewStatus()` to reconstruct before/after logic.
3. For each `ProcessEventListener`: add new v9 methods (`onSignal`, `onMessage`, `onError`, etc.) as needed.
4. Annotate listeners with `@ApplicationScoped` for CDI auto-registration.
5. Configure Kafka in `application.properties` (topics, connectors, serializers).
6. Remove any v8 KIE Server Kafka emission configuration (`kieserver.kafka.*` properties).
7. Run `mvn clean package` and smoke-test event emission.

### Knowledge Sources
> All conversion steps in this sub-task must be grounded against the following official BAMOE 9.5.x
> documents. Consult the linked source before modifying any event listener or Kafka configuration.

| Step | Authoritative source |
|---|---|
| Add process-event / messaging add-on dependencies | [`upgrade/05-09-upgrading-event-streaming.adoc` — Messaging event add-on / Process event add-on](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-09-upgrading-event-streaming.adoc) |
| `TaskLifeCycleEventListener` → `UserTaskEventListener` migration | [`upgrade/05-09-upgrading-event-streaming.adoc` — The User Task Event Listener Interface](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-09-upgrading-event-streaming.adoc) |
| `ProcessEventListener` new methods (`onSignal`, `onMessage`, etc.) | [`upgrade/05-09-upgrading-event-streaming.adoc` — The Process Event Listener Interface](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-09-upgrading-event-streaming.adoc) |
| CDI `@ApplicationScoped` registration | [`upgrade/05-09-upgrading-event-streaming.adoc` — CDI registration](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-09-upgrading-event-streaming.adoc) |
| Kafka topic / connector `application.properties` | [`upgrade/05-02-upgrading-client-server-projects.adoc` — application.properties Kafka block](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/05-02-upgrading-client-server-projects.adoc) |
| Event-driven workflows reference | [`workflow/event-driven-workflow.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/workflow/event-driven-workflow.adoc) |
| Tutorial: event listener migration end-to-end | Tutorial 05 in the BAMOE v8→v9 migration tutorials repo (linked from `upgrade/05-09-upgrading-event-streaming.adoc`) |

---

## Sub-Task 10 — Runtime Deployment and Live-Data Strategy

**Status:** `[ ] pending`

### Intent
Deploy the migrated v9 service to the target runtime (local, OpenShift, or Kubernetes) and define
the strategy for handling in-flight v8 process instances during cutover.

### Expected Outcomes
- Service deployed and accessible; Swagger UI reachable at `/q/swagger-ui/#/`.
- BAMOE Management Console deployed (Helm chart) and connected to the Business Service.
- Live-data cutover strategy documented and agreed with stakeholders:
  - New database provisioned for v9 (schemas are incompatible — binary persistence format changed).
  - v8 and v9 environments run in parallel.
  - New process instances routed to v9; existing in-flight instances remain on v8 until completion.
  - v8 decommissioned once all active instances finish.
- No automated data migration tool is available from IBM; the side-by-side approach is the
  only supported strategy.

### Todo List
1. Configure persistence in `application.properties`:
   - Provision a new PostgreSQL (or SQL Server / Oracle) database — **do not reuse the v8 database**.
   - Set `kogito.persistence.type=jdbc`, datasource URL, username, password.
   - Set `quarkus.hibernate-orm.database.generation=update` (or equivalent Spring Boot property)
     for initial schema creation.
2. Deploy the v9 service:
   - **Local / Dev Mode:** `mvn clean quarkus:dev`; verify at `http://localhost:8080/q/swagger-ui/#/`.
   - **OpenShift:** follow `getting-started/deploying-to-openshift.adoc`; use `mvn clean package -Dquarkus.container-image.build=true` to build the container image.
3. Deploy BAMOE Management Console using the Runtime Environment Helm chart;
   see `installation/runtime-environment.adoc`.
4. Smoke-test all process definitions and REST endpoints against the v9 Swagger UI.
5. Agree and document the parallel-run cutover plan with operations and business stakeholders.
6. Set up traffic routing: new process start requests → v9; existing instance callbacks → v8.
7. Monitor v8 for instance completion; decommission once the active instance count reaches zero.

### Knowledge Sources
> All deployment and cutover decisions must be grounded against the following official BAMOE 9.5.x
> documents. Do not assume v8 runtime behaviour carries over.

| Step | Authoritative source |
|---|---|
| Database schema incompatibility; side-by-side strategy | [`upgrade/07-live-data-considerations.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/07-live-data-considerations.adoc) |
| v8 → v9 runtime architecture comparison | [`upgrade/06-production-environment-comparison.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/06-production-environment-comparison.adoc) |
| Deploying a Business Service to OpenShift | [`getting-started/deploying-to-openshift.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/getting-started/deploying-to-openshift.adoc) |
| Management Console Helm chart installation | [`installation/runtime-environment.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/installation/runtime-environment.adoc) |
| Management Console features vs Business Central | [`managing-monitoring/consoles.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/managing-monitoring/consoles.adoc) |
| Operational considerations post-cutover | [`upgrade/08-operation-considerations.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/08-operation-considerations.adoc) |
| Performance considerations | [`upgrade/09-performance-considerations.adoc`](https://raw.githubusercontent.com/IBM/bamoe-docs/9.5.x/upgrade/09-performance-considerations.adoc) |

---

## Key Decisions and Constraints

| Topic | v8 | v9.5 |
|---|---|---|
| Java version | 8 or 11 | **21 (required)** |
| Maven version | 3.6.x | **3.9.11+** |
| Runtime framework | JBoss EAP / WildFly | **Quarkus 3.x or Spring Boot 3.x** |
| Authoring tool | Business Central | **BAMOE Canvas + Developer Tools for VS Code** |
| Execution model | KIE Server (KJAR, centralised) | **Kogito microservice (JAR / container image)** |
| Configuration | `kmodule.xml`, `kie-deployment-descriptor.xml`, `persistence.xml` | **`pom.xml` + `application.properties` only** |
| Security | WildFly JAAS / LDAP | **OIDC (Keycloak or any compliant IdP)** |
| Database | Multiple incl. Sybase/EDB | **PostgreSQL, SQL Server, Oracle only** |
| PMML | Supported | **Not supported — must replace** |
| Guided Rules / DSL | Supported | **Must convert to DRL or DMN** |
| JavaScript in BPMN | Supported | **Not supported — must use Java** |
| Forms | Legacy Form Modeler `.frm` | **Regenerate via Developer Tools** |
| Shared KIE sessions (BPMN+DRL) | Supported | **Not supported — refactor to data mapping** |
| In-flight data migration | N/A | **Side-by-side parallel run only (no tooling)** |
