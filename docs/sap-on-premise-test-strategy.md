# SAP On-Premise Holistic Test Strategy

## 1. Purpose and status

This concept defines the target operating model for a business-process-oriented, risk-based test strategy in an SAP on-premise landscape. Some unit, API, and manual tests already exist. The goal is to connect and improve that investment, fill material coverage gaps, and make release decisions traceable to exact transport content and observed business outcomes.

The delivery route is **DEV (Development) -> INT (Integration Test) -> ACC (Acceptance) -> PRD (Production)**. STMS remains the transport mechanism. INT discovers integration problems; ACC qualifies a stable, production-bound release candidate; PRD verifies deployment and relevant business-cycle outcomes.

This is a strategy, not a record of completed configuration or implemented integrations. The stated landscape and tool allocation are the planning context. Installed versions, editions, licenses, connectors, APIs, configuration, and end-to-end operation have not been verified by this document. Public product references in section 15 describe capabilities or examples, not proof that they are available in this installation. Gate enforcement, orchestration, report adapters, reconciliation, and approval snapshots described here are proposed controlled workflows unless explicitly provided and verified in the installed products.

### Principles

- Retain useful existing tests; review, improve, or retire them based on purpose, reliability, ownership, and risk. Do not rebuild everything or pursue blanket UI automation.
- Cover custom code, customizing, cross-module processes, roles, interfaces, batch jobs, financial effects, and operational readiness. Include security, performance, and recovery according to risk.
- Prefer many fast, focused checks and fewer end-to-end UI tests. Validate persistent business state, downstream documents, postings, and reconciliation, not just HTTP 200, a visible screen, or a clean import.
- Reuse test definitions across environments and releases, but never assume that results from one baseline prove another baseline.
- An automated pass is not business acceptance. A successful transport import is not functional success.
- Begin with one critical process, such as order-to-cash, procure-to-pay, closing, or manufacturing, and expand using evidence from the pilot.

## 2. Risk-based scope and existing-test inventory

For each change, assess business criticality, implementation complexity, dependencies, affected scope, and reversibility. Record a low, medium, or high risk classification in Jira, with the rationale and affected process variants. A small customizing change can be high risk if it changes tax, account determination, payments, authorization, or shared behavior.

| Risk | Typical exposure | Qualification approach |
|---|---|---|
| Low | Localized, understood behavior with limited dependencies and a proven recovery path | Focused developer and component checks, affected integration checks, and selected critical smoke |
| Medium | Multiple variants or components, interfaces, roles, or customizing with meaningful downstream effects | Broader integration, negative and regression scenarios, plus relevant ACC acceptance and operational checks |
| High | Critical financial or operational outcomes, broad dependencies, difficult reversal, or high uncertainty | Critical end-to-end and variant coverage, business acceptance, deployment rehearsal, and applicable security, performance, and recovery evidence |

These are scope recommendations, not automatic exemptions from the release gates. The test lead and process owner agree mandatory scenarios and variants before execution. Escalate risk when the impact is uncertain; do not infer low risk from transport size.

Inventory current tests with their process objective, requirements, layer, tool, owner, reference, execution history, reliability, and data/environment dependencies. Assign a disposition:

| Disposition | Action |
|---|---|
| Retain | Keep a relevant, reliable test and connect its definition and evidence to the operating model |
| Improve | Repair weak assertions, unstable data, missing variants, poor diagnostics, or unclear ownership |
| Retire | Record why a redundant, obsolete, or ineffective test is no longer needed and assess any resulting gap |
| Add | Address an uncovered material risk at the lowest effective test layer |

For legacy code, add tests for new or changed logic, recurring defects, and high-risk behavior. Do not require a blanket retrofit of all existing code or an arbitrary universal code-coverage target.

## 3. Tool responsibilities and sources of truth

| Tool or repository | Authoritative responsibility | Boundary |
|---|---|---|
| Confluence | Published strategy and policy, process catalogue, landscape charters, risk model, data/environment standards, gate policy, release and recovery playbooks | Link executable tests rather than copying their steps. This repository holds the version-controlled strategy concept; publishing changes must be controlled so competing policy copies do not emerge |
| Jira | Requirements and acceptance criteria, change risk, defects, exceptions, release manifests, business and technical approvals | Workflow status alone is not release readiness |
| Xray in Jira | Reusable test definitions, scoped plans, executions/results, and requirement coverage; authoritative test-evidence index | Not the execution engine and not an inherent STMS import lock |
| Bruno in Git | HTTP-facing API collections, assertions, and versioned automated implementation; CLI execution and JUnit/JSON/HTML reporting | Does not directly run RFC or IDoc paths; report support must be checked against the pinned CLI version |
| suxxesso | SAP GUI/Fiori process automation, reusable components, parameterized plans, and detailed execution artifacts | Use only verified capabilities of the installed Test Cockpit, Suite Server, and Data Manager; no connector or unattended API is assumed |
| ABAP Unit and ATC | Developer-level behavior checks and static assurance | Do not replace integration or acceptance evidence |
| STMS | Transport requests, routes, queues, ordered import content, and import logs | A clean import does not certify business behavior |

Keep structured manual executable steps in Xray, not duplicated in Confluence. Keep automated implementation in Bruno or suxxesso, with an objective and stable implementation reference in Xray. Detailed reports may remain in controlled artifact storage, but Xray must retain their durable references and qualification context.

## 4. Test layers and landscape charters

Each environment needs a charter defining its owner, software/configuration baseline, clients, roles, interface destinations, production differences, allowed changes, scheduling, and readiness criteria.

| Stage | Purpose and test scope | Operating constraints |
|---|---|---|
| DEV | Review and ATC; ABAP Unit; developer component/API checks for behavior, validation, persistence, authentication/authorization, and failure handling | Run during development and before release; link selected results and native unit/ATC evidence rather than importing every ABAP method into Xray |
| INT | Real test integrations where feasible; cross-module and contract tests; negative paths, retries, duplicates, jobs, locks, backlogs, and reconciliation; exploratory testing and risk-based regression | Coordinate frequent imports and record the baseline of each batch/cycle. Use asynchronous correlation and bounded polling rather than unbounded waits |
| ACC | Critical end-to-end regression; changed company-code, plant, role, country, and document-type variants; manual business acceptance; deployment rehearsal and applicable nonfunctional/operational checks | Qualify a stable candidate with production-representative data, roles, and configuration. Freeze candidate-only windows and prevent unrelated imports or testing against a changing state |
| PRD | Import-log assessment; safe smoke; interface, job, monitoring, and reconciliation checks | Prefer read-only checks, or narrowly scoped, explicitly approved actions. Observe relevant overnight, period-end, or business-cycle outcomes before closing the release |

Component/API checks should test business rules close to their implementation. INT should prove cross-component behavior, including error propagation, retry safety, duplicate handling, and eventual business consistency. ACC should prove the intended production combination, not repeat every lower-layer test without a risk rationale.

Bruno can exercise HTTP-facing OData, SOAP, and other HTTP services. Direct RFC, IDoc, and other non-HTTP interactions require suitable tools or harnesses. An HTTP wrapper can initiate a journey but does not prove the complete downstream path without observations and reconciliation.

Design suxxesso automation from reusable business components and explicit assertions instead of giant recordings. Parameterize relevant data and variants while preserving independently assessable objectives.

For a journey crossing tools, pass an explicit run ID and business document IDs between stages. Either use separate Xray Tests for separately assessable objectives or one orchestrator that aggregates all required stages into a single outcome. Never let several tools independently overwrite the same Test Run. A request or assertion is not automatically a distinct business scenario.

Performance/load, specialist security, and recovery exercises require appropriate methods and tools; API response timing and UI automation do not replace them. Run disruptive work in a suitable isolated environment or reserved window. Define workload, expected outcomes, recovery objectives, and observation responsibilities before execution.

## 5. Xray test model and organization

Use native Xray entities instead of homemade Jira tasks for each test execution.

| Entity | Meaning and intended use |
|---|---|
| Test | A reusable, independently assessable scenario with an objective and expected outcome |
| Manual Test | Structured actions, data, and expected results for manual functional testing or UAT |
| Generic Test | An externally automated Bruno or suxxesso scenario; GUI automation is not a Manual Test |
| Cucumber Test | Use only where an actual Gherkin implementation exists |
| Precondition | Reusable prerequisites; it does not automatically provision data or environments |
| Test Set | Reusable, potentially overlapping selection such as critical OTC regression, authorization, or smoke |
| Test Plan | Scoped release-stage and candidate qualification |
| Test Execution | One execution session against a known baseline |
| Test Run | A Test's result within a Test Execution, carrying results, evidence, and defect links; not a separate Jira issue |

Organize Test Repository folders by business process: order-to-cash, procure-to-pay, finance/closing, manufacturing, and cross-process controls. Use Test Sets for overlapping selections instead of duplicating a second hierarchy.

Test metadata should include process ID, test level, criticality, tool, owner, stable automation reference, applicable variants, and lifecycle. Maintain a definition lifecycle such as draft, reviewed, active, and retired separately from execution outcomes, using supported configuration.

Reuse Tests across INT, ACC, and releases; do not create duplicates merely because the environment changes. Create distinct Tests when the objectives genuinely differ. Record required variants explicitly rather than relying on a test title to imply them.

Configure the relevant Jira requirement issue types as Xray-coverable and use the designated Xray coverage relationship. Generic "relates to" links are not a substitute. A coverage link alone does not demonstrate that every acceptance criterion or variant has been verified.

### Plans, environments, and candidate baselines

Link separate INT qualification and ACC qualification Test Plans to the same Jira release. INT uses Test Executions per import batch or cycle. ACC uses an explicit qualification plan for each candidate revision, retaining historical plans. Use a small PRD verification plan for deployment and operational evidence.

At the ACC scope freeze, capture the explicit Test list, required variants, definition versions, and versioned transport manifest. Use static membership, not a filter whose results silently change. Dynamic-plan availability depends on the Xray edition/license and must not undermine the frozen qualification scope. Where native definition versioning or scope snapshots are insufficient, preserve controlled exports/references.

A revised candidate receives a new ACC qualification plan. Carry forward scope, not assumed passes. Reference unchanged prior evidence only through a documented, approved impact assessment naming the evidence, its original baseline, the current scope it supports, and why it remains applicable. Do not fabricate new runs or present old outcomes as new execution.

Use controlled Xray Test Environment values such as `DEV-100`, `INT-100`, `ACC-100`, and `PRD-100`. Include SAP SID in the naming convention or metadata wherever necessary to avoid ambiguity. A Test Execution must identify exactly one system/client baseline; never label it both INT and ACC to represent a route.

Record role, plant, company code, and data variants explicitly through supported iterations or separate executions. Account for every required variant, including failures, omissions, and blocked iterations.

Each Test Execution needs:

- Jira release and candidate revision, plus the reference to the versioned, ordered manifest.
- System/SID and client, confirmed import-finish timestamp, and any documented baseline differences.
- Test-definition versions, Bruno collection commit and runner version, or suxxesso scenario/suite version.
- Dataset references, required variants, external run ID, and relevant business document IDs.
- Execution owner and timestamps, retained report links, outcome details, and linked defects.

Use native fields where possible and only the minimum custom fields needed. Jira Fix Version identifies the release, not a unique transport state. Xray's latest or consolidated status is not a release certificate: qualification must select the correct plan, environment, candidate, and qualifying executions. Do not assume all custom-field filtering is available natively; use a controlled supplementary report where needed.

## 6. Transport candidates and release manifests

Maintain a versioned candidate record under the Jira release. Each candidate contains:

| Area | Required content |
|---|---|
| Identity and scope | Release, unique candidate revision, included changes, risk assessments, affected processes and variants |
| Deployment content | Ordered workbench and customizing transports, dependencies, target systems/clients, manual and nontransport steps |
| Qualification | Frozen Tests/variants/definition versions, script versions, execution links, baseline context, defects and approved exceptions |
| Decision and operation | Business and technical approvals, authorized release decision, deployment sequence, monitoring and recovery plan |

Assess overlapping objects, overtaking, and import order, not only membership in a list. INT may contain changes that are not going to PRD with this candidate; identify dependencies on those changes and qualify the actual production combination in ACC.

ACC should start from a production-aligned baseline plus the intended candidate. Document known differences and their qualification impact. Removing a transport from the manifest does not undo its already imported effects: apply a controlled correction or reset to restore the intended ACC state, then reassess and requalify.

Reconcile transports of copies used for testing with the final production requests, including content and order. Multiple SAP clients do not isolate shared repository objects; client separation is not enough to protect a candidate from unrelated workbench changes.

Any candidate change reopens impact assessment, affected gates, and approval. Document impact-based retesting; rerun directly affected cases and critical smoke. Broaden regression where dependencies or uncertainty warrant it.

## 7. Quality gates and decisions

The gates are a recommended organizational control model. They are not automatically enforced by Jira, Xray, or STMS.

| Gate | Required evidence and decision |
|---|---|
| G0: Ready for development | Acceptance criteria, risk, affected processes, data needs, owners, and intended test scope agreed |
| G1: DEV -> INT | Review, applicable ATC/ABAP Unit checks, developer tests, dependencies and prerequisites complete; development ownership identified |
| G2: INT -> ACC | Candidate defined; affected-combination integration evidence, regression, negative, job, and interface outcomes satisfactory; no acceptance-blocking defects; stable ACC baseline, roles, data, and window ready; test lead recommends and release owner authorizes |
| G3: ACC -> PRD | Final manifest imported and tested in intended order; all selected mandatory scenarios and variants complete and passing; blockers resolved; explicit permitted exceptions recorded; separate business and operational/technical approval; recovery and monitoring ready; authorized release owner approves |
| G4: PRD verification and closure | Basis reviews import logs and warnings; safe smoke and relevant operational reconciliation complete; required business-cycle observation finished before release closure |

ATC transport-release checks depend on SAP release and configuration. They do not supply all business gates. Initially restrict STMS permissions and have Basis verify the documented Jira approval before importing. Supported automation or validators may later strengthen the control, but require a separately designed and verified implementation.

### G3 decision rules and approval snapshot

Do not use an arbitrary 95% pass threshold or Jira "Done" as the release criterion. Failed, blocked, TODO, not-run, missing, stale, wrong-environment, and incomplete-evidence results do not pass. A technical automation pass does not replace the business owner's acceptance.

An exception must identify its exact scope and risk, justification, compensating controls, owner, authorized approvers, expiry or review date, and follow-up action. It is an explicit departure under the exception policy, not a relabeling of a failure as pass. Define which blockers cannot be excepted; no unresolved blocker is implicitly waived.

Retain an approval snapshot with the exact manifest revision, frozen scope/variants/versions, qualifying runs and outcomes, approved prior-evidence assessments, defects/exceptions, approvers, and timestamps. Use controlled storage or exports with access and retention policies. Do not claim that Jira/Xray records or attachments are immutable by default.

### Failure, emergency, and recovery paths

The normal defect route is: INT or ACC failure -> Jira defect -> fix in DEV -> verification in INT -> revised ACC candidate -> affected regression and renewed approval. Do not perform direct transported-code or customizing repairs in ACC.

Preserve DEV -> INT -> ACC -> PRD for emergency changes wherever feasible, with a compressed, risk-justified scope. Any bypass requires explicit authorization, compensating evidence, retrospective testing, and baseline reconciliation. An emergency fix overlapping an ACC candidate requires object/dependency impact assessment and requalification of that candidate.

There is no assumed simple transport rollback. Recovery can require a corrective transport, configuration reversal, business-data repair, or coordinated restore, depending on the effects and consistency requirements. Assign decision authority and validate the applicable recovery playbook before deployment.

For G4, Basis must assess RC4 warnings rather than automatically accepting them; import errors prevent closure until resolved. Keep the release deployed but not closed until relevant overnight jobs or business-cycle reconciliations finish.

## 8. Execution and evidence integration design

### Start with controlled manual orchestration

After Basis confirms STMS import completion and environment readiness, create a scoped Xray Test Execution with the known baseline, run the selected tools or manual scenarios, and attach, import, or link the evidence. Reconcile scope and results before considering the execution complete.

Only then introduce an existing internal scheduler or CI runner and supported interfaces. Importing results into Xray does not inherently start Bruno or suxxesso. Verify installed Xray deployment type, version, edition, license, authentication, endpoints, result formats, and feature support before implementation. Xray Cloud and Data Center interfaces differ; Cloud evidence egress must satisfy security policy.

### Bruno to Xray

1. Create the intended Test Execution with planned existing Tests, required variants, and complete baseline context.
2. Run a pinned Bruno collection commit with a pinned CLI version. Inject secrets securely; never commit them or include them in reports.
3. Generate a JUnit report and retained detailed output as supported by the pinned runner. Define whether the assessable scenario contains multiple requests/assertions; every required assertion must pass.
4. Map results explicitly and stably to existing Xray Test keys. For Xray Cloud, documented JUnit behavior uses `classname` and `name` to identify Generic definitions unless an explicit supported Test key/ID mapping is supplied; unmatched results can create new Tests.
5. Where necessary, use a small validated adapter to enrich the report with the supported mapping, or use Xray JSON. This is proposed integration work, not an already available Bruno-to-Xray connector. Verify the format against the installed Xray product.
6. Import into the intended execution, confirm ingestion completion, then reconcile planned versus reported Tests, variants, outcomes, and evidence. Reject unknown, duplicate, or missing mappings for qualification.

Renaming a request or report item must not silently create a duplicate Test. Maintain a controlled mapping between stable scenario identity and Xray Test key. The documented Xray Cloud JUnit import maps skipped tests to TODO; neither a successful import nor an otherwise green report proves that all required tests ran. Reconcile expected scope with actual outcomes after every import.

### suxxesso to Xray

A Generic Test references a stable suxxesso scenario. Select automation through a maintained mapping and use supported execution/export interfaces, or a native connector only if verified for the installed version and license. Otherwise design and validate a documented export adapter to an Xray-supported result format. Do not assume a native Jira connector, an unattended API, or a particular server-side execution feature exists.

Assess Test Cockpit for documented test-management/execution roles, Suite Server for its documented suite/server roles, and Data Manager for its documented data-selection/management roles. Confirm the exact division of responsibilities in the installed suite before assigning unattended operations.

Capture the result, timestamps, run ID, selected dataset, relevant document IDs, failure details, appropriate screenshots, and durable full-report links. Data Manager selection does not replace data masking, setup, reservation, or cleanup. Record the actual selected data and versions.

### Reruns, transmission retries, and failure categories

A new actual test run creates a new Test Execution. Retain the earlier failure, evidence, and defect; do not rerun until green and hide the sequence. A transmission retry of the same external run uses the same mapping and idempotent handling, without creating duplicate executions. Idempotent handling must be verified as part of the integration, not assumed from an import endpoint.

Distinguish application failure, blocked prerequisites, automation/infrastructure error, and import/evidence failure. Use supported statuses plus a supplementary category where necessary; do not assume custom statuses exist. None of these conditions can qualify as success.

Track ingestion completion and evidence completeness separately from tool execution. A partial suite, a stale report, a missing stage, a timeout, or a successful upload with incomplete results must remain nonqualifying. Surface failures explicitly and route them to the appropriate owner.

## 9. Test data, environments, and evidence controls

INT needs broad normal, edge, and failure datasets. ACC needs stable, representative scenarios matching the candidate's business variants. Use synthetic data or masked production-derived data with least-privilege access, retention rules, and a documented refresh/setup process.

Reserve datasets or allocate unique run/document IDs to avoid collisions. Define cleanup and business reversal procedures, including interrupted-run recovery. Respect posting periods, date-dependent behavior, overnight processing, and period-end schedules.

Use real test interface endpoints wherever feasible. Controlled virtualization is useful for faults and unavailable dependencies, but must be labeled. Record known ACC integration gaps and compensating INT evidence; a stubbed path is not a fully tested production path.

Prevent real payments, emails, shipments, and other unintended external actions from nonproduction systems. Verify destination controls and approvals, not just environment labels.

For GUI automation, define host and session prerequisites, authorizations, supported concurrency, scheduling, and interrupted-session recovery. Ensure parallel tests cannot consume or corrupt each other's data.

Redact sensitive data from reports, screenshots, and document references as necessary. Credentials must never enter Git, Jira, Confluence, logs, or reports. Keep artifact links durable and versioned, with access controls and retention aligned to the release/evidence policy. Regulatory or audit controls require explicit validation of the configured process; product marketing or a report link is not proof of compliance.

## 10. Ownership and segregation of duties

| Role | Accountability |
|---|---|
| Business process owner | Business risk, expected outcomes, UAT, and residual-risk acceptance |
| Functional specialists | Process and customizing impact, variants, representative data, functional expected results |
| Development | ABAP Unit, API/component tests, ATC, code review, and fixes |
| Test lead | Scope, coverage, plan integrity, execution coordination, evidence reconciliation, qualification recommendation |
| Basis | Environments, import readiness, transport sequence, STMS permissions/imports/logs, baseline control |
| Operations/security specialists | Monitoring, authorization/security, performance, and recovery checks in their disciplines |
| Release owner | Final authorized release decision, gate completion, exceptions under policy, and coordination of closure |

Roles may overlap in a small team, but preserve required segregation of duties. Separate business acceptance from operational/technical approval even when one person has multiple responsibilities; record the capacity in which each decision was made.

## 11. Readiness reporting and improvement metrics

Report readiness for the exact candidate, not merely a release-wide or latest-test aggregate. Show the frozen scope and denominator, required variants, qualifying evidence, nonqualifying states, open defects, exceptions, and approval status.

| Metric | Decision it supports |
|---|---|
| Requirements without reviewed Tests | Find unexamined acceptance and coverage gaps |
| Mandatory scenario/variant scope versus actual execution | Reveal missing, blocked, failed, or incomplete qualification |
| ACC candidate readiness | Determine whether the exact candidate meets G3 |
| Failing/blocked scenarios and linked defects | Prioritize remediation and accountable owners |
| Production changes with complete evidence | Assess adherence to transport-linked governance |
| INT -> ACC and ACC -> PRD defect escapes | Improve where defects should have been detected |
| Change failure rate | Relate test effectiveness to actual release outcomes |
| Regression duration and environment blockage | Reduce release delay without weakening scope |
| Flaky tests and unmapped/duplicate result imports | Improve trust in automation and evidence |
| Candidate churn and exception age | Expose unstable qualification and accumulating residual risk |

Agree precise definitions and numerical targets after collecting a baseline. Coverage links do not prove complete acceptance criteria or variant coverage. Do not use a blanket code-coverage or automation-percentage target as a release certificate.

## 12. Phased rollout: approximately 90 days

The timing is indicative and depends on access, tool compatibility, and environment readiness. Each phase delivers an operating capability, not just a collection of scripts.

| Phase | Focus and exit evidence |
|---|---|
| Days 1-30: Foundation | Inventory and disposition existing tests; select one critical process; agree risks, owners, landscape charters, data standards, and gate policy; define manifest/approval templates; configure and verify Xray requirement coverage, repository structure, metadata, environments, and entity usage |
| Days 31-60: Process pilot | Create reviewed Tests, Preconditions, and Test Sets; use separate INT/ACC plans; validate Bruno result import and a supported suxxesso export/integration path; include manual ACC UAT; use a versioned manifest and approval snapshot |
| Days 61-90: Operationalize and expand | Complete one normal release and a failed-test/defect/DEV-fix/INT-retest/revised-ACC-qualification cycle; strengthen evidence automation with the existing runner where justified; measure outcomes and expand the next critical scope |

Before scaling, demonstrate that the qualification workflow rejects missing results, wrong environments, stale candidate passes, duplicate or unknown mappings, skipped tests, partial runs, and incomplete evidence. Demonstrate that a transmission retry is idempotent and a genuine rerun preserves prior failures in a separate execution. Verify that all required variants and cross-tool stages are accounted for.

The pilot must verify tool capabilities rather than assume them. If an unattended integration is unavailable, retain the controlled manual execution/export workflow with reconciled evidence; do not claim that writing this strategy implemented an integration.

## 13. Minimum release qualification record

Use the following as a compact review checklist, implemented through native records and minimal supplementary fields rather than duplicate per-test tasks:

| Record | Required review question |
|---|---|
| Candidate and baseline | Is the exact production-bound combination known, ordered, and reconciled to the tested ACC state? |
| Scope | Are all mandatory Tests, variants, definition/script versions, and approved impact assessments frozen and identifiable? |
| Executions | Does every qualifying outcome belong to the intended system/client and candidate, with complete retained evidence? |
| Defects and exceptions | Are blockers resolved and all permitted departures explicit, owned, time-bounded, and approved? |
| Acceptance | Are business approval and operational/technical approval independently recorded? |
| Deployment and recovery | Are Basis authorization, manual steps, monitoring, recovery decisions, and safe PRD checks ready? |
| Closure | Have import warnings/errors been assessed and required business-cycle reconciliations completed? |

## 14. Capability-verification boundary

Before automating any part of this model, record the installed version/edition/license, relevant vendor documentation, supported interface, authentication model, data destination, and the observed pilot outcome. At minimum verify:

- Xray Cloud versus Data Center behavior, requirement coverage configuration, result formats, explicit Test/Execution mapping, status/skip semantics, iterations, plan membership, reporting filters, and artifact retention.
- Bruno CLI report generation, scenario granularity, runner pinning, error exit behavior, and mapping preservation across renames.
- suxxesso execution/export options, supported SAP GUI/Fiori scope, component and parameterization features, server/session prerequisites, Data Manager behavior, and any claimed connector or unattended interface.
- SAP ATC transport-release behavior and STMS permissions in the configured SAP release.
- Custom orchestration, ingestion reconciliation, idempotency, gate controls, and snapshot storage through positive and negative pilot cases.

No integration is considered verified merely because a product supports an adjacent capability. No Jira approval is considered an enforced STMS lock without a separately implemented and verified control.

## 15. Product references

These are product references, not evidence of installed capabilities. The Xray links below are **Cloud examples**; use matching Data Center documentation when the installation is Data Center, and validate edition/license and version differences before adopting any example.

| Reference | Intended use |
|---|---|
| [Xray Cloud: Test Plan](https://docs.getxray.app/display/XRAYCLOUD/Test+Plan) | Plan model and available scoping behavior |
| [Xray Cloud: Test Environments](https://docs.getxray.app/display/XRAYCLOUD/Test+Environments) | Environment-specific execution and reporting concepts |
| [Xray Cloud: Taking advantage of JUnit XML reports](https://docs.getxray.app/display/XRAYCLOUD/Taking+advantage+of+JUnit+XML+reports) | Test identification/mapping, result import, and skipped-result semantics |
| [Xray Cloud: Import Execution Results - REST](https://docs.getxray.app/display/XRAYCLOUD/Import+Execution+Results+-+REST) | Supported result-import interfaces and formats |
| [Bruno CLI overview](https://docs.usebruno.com/bru-cli/overview) | CLI execution and reporting entry point |
| [suxxesso Test Cockpit](https://www.suxxesso.com/en/tool-suite/test-cockpit/) | Published Test Cockpit scope; verify installed capabilities |
| [suxxesso Suite Server](https://www.suxxesso.com/en/tool-suite/suite-server/) | Published Suite Server scope; verify installed capabilities |
| [suxxesso Data Manager](https://www.suxxesso.com/en/tool-suite/data-manager/) | Published Data Manager scope; verify installed capabilities |

## Target operating model

**Xray is authoritative for test definitions and test evidence; INT provides integration discovery; ACC provides stable, auditable release qualification; STMS performs approved deployment.** Jira connects risk, requirements, manifests, defects, exceptions, and approvals; Confluence publishes the governing policy and playbooks. Bruno, suxxesso, ABAP Unit, and ATC produce complementary evidence under verified operating procedures. PRD verification and business-cycle observation complete the release, rather than a clean import or an isolated green result.
