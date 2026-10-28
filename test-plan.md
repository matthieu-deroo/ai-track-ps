# RetroFlow: Risk-Based QA Test Plan

## Purpose and scope

This plan focuses on whether RetroFlow can reliably preserve team retrospective records
and action-item progress before it is rolled out to StackCraft's 12 teams. The system is a
Spring Boot REST API with an in-memory H2 database and no authentication. Test the API
contract and the business rules described in the handover brief; do not treat the existing
developer tests as proof that a rule works.

The plan prioritizes record integrity and irreversible state changes first, then workflow
constraints, then retrieval and input validation. Frontend, full penetration, load testing,
and the later User Directory integration are out of scope. Basic startup and API sanity
checks are included.

## Test levels and why

| Level | How it will be used | Why it matters |
|---|---|---|
| **API-level (primary)** | Send real HTTP requests to a running application. Check status codes, response bodies, and persisted state after both allowed and rejected operations. Automate the four core rules: only one open retrospective per team, no reopening after close, no feedback changes/deletes after close, and no uncompleting a completed action. | The release risk is what clients can actually do through the API. These tests exercise routing, request validation, business logic, persistence, and serialization together. |
| **Integration** | Run the application with its Spring/JPA wiring and H2 database. Verify relationships and state transitions across multiple requests; restart or reset the test context as appropriate to confirm test isolation. | A controller-only or mocked test can miss persistence, transaction, mapping, and database constraint defects. Integration coverage helps verify that state changes survive between requests within the running service. |
| **Exploratory** | Follow charters for boundary inputs, invalid references, repeated requests, alternate request order, and combinations of operations. Record exact requests and responses for defects. | The brief says validation is incomplete and the inherited suite was written under time pressure. Exploration is useful for discovering unspecified edge cases and incorrect assumptions that scripted happy paths may miss. |

These levels are complementary: API tests provide release-facing evidence, integration
tests help identify failures in the persistence path, and exploratory testing probes
behavior not yet represented by reliable automation.

## Entry criteria

Begin execution when:

- The application can be built and started using the project's documented instructions.
- The API base URL, endpoint documentation or API reference, and expected request shapes
  are available.
- A disposable test environment is available, with permission to create and delete test
  records and no production data.
- The test team can inspect responses and application logs when a result is unexpected.
- A repeatable way to reset or isolate test data is agreed. If the H2 database is in
  memory, confirm how the application and test client share its lifetime.

If any prerequisite is missing, record it as a blocker or limitation rather than assuming
the behavior is correct.

## Exit criteria

Finish this QA pass when:

- Each in-scope business rule has been exercised at least once through the API, including
  a successful case and the relevant rejected or boundary case.
- Automated API tests for all four core rules have run against the live application, and
  their results are recorded. A failing test is investigated, not simply deleted or
  weakened.
- Exploratory findings have reproducible requests, actual responses, and business impact.
- Findings are triaged by severity, with known limitations and untested areas stated.
- The release recommendation is documented as **BLOCKED**, **CONDITIONAL**, or
  **APPROVED**, based on observed evidence. Any unresolved defect that can corrupt or
  misrepresent records must be explicitly considered before approval.

Passing these criteria does not mean full coverage; it means the agreed high-risk scope
has evidence and remaining uncertainty is visible.

## Risk scale and prioritization

Use a 1–3 scale for both likelihood and impact. The risk score is likelihood multiplied
by impact. The scores are a planning estimate, not measured defect probabilities.

**Likelihood:** 1 = less likely; 2 = plausible; 3 = more likely. The application is a
first version with no proper QA yet, and the brief specifically warns that validation may
be incomplete, so no rule is assigned the lowest likelihood without evidence.

**Impact:** 1 = limited inconvenience; 2 = significant workflow disruption or misleading
status; 3 = loss or corruption of the trusted retrospective record, or an irreversible
business-state error.

| Priority | Business rule / behavior | Likelihood | Impact | Score | Why this priority |
|---:|---|:---:|:---:|:---:|---|
| **1** | Feedback in a closed retrospective cannot be modified or deleted. | 3 | 3 | **9** | This is the strongest record-integrity safeguard. A silent edit or deletion can erase concerns or commitments after the team believes the discussion is final, undermining the historical record across teams. Test update and delete separately, then retrieve the retrospective to verify the original item remains unchanged. |
| **2** | Concurrent feedback from multiple team members is persisted once with the correct content and submitter; a feedback/close race preserves a valid final state. | 2 | 3 | **6** | Lost or duplicated feedback or incorrect attribution would make the shared retrospective record incomplete or misleading. A race with closure could also allow feedback to be appended after finalization. Test parallel submissions and race a submission against closing, then retrieve and verify all resulting records. |
| **3** | A closed retrospective cannot be reopened. | 2 | 3 | **6** | Reopening permits activity against a record that should be final and can weaken the protection of closed feedback. The state transition is one-way by definition, so test attempts to reopen through every available status-changing route and verify the status remains closed. |
| **4** | A completed action item cannot be uncompleted. | 2 | 3 | **6** | Reversal can falsely show that agreed work is outstanding or allow completion history to be rewritten. This is a one-way state rule; test both the completion operation and any general update route that could alter the flag. |
| **5** | A team may have only one open retrospective at a time. | 3 | 2 | **6** | The rule is central to a coherent team workflow and may be missed in request validation or concurrent creation. Multiple open records could split discussion and feedback. Test sequential creation and, if time permits, near-simultaneous creation for the same team. |
| **6** | Feedback may be added only to an open retrospective. | 2 | 2 | **4** | Accepting late feedback after closure conflicts with the final-record boundary and can make retrospective history inconsistent. Test adding to both open and closed records. |
| **7** | A team must have at least one member. | 2 | 2 | **4** | A memberless team cannot use the intended team-member workflow and may leave unusable records. Test empty and omitted member lists, plus a valid one-member team. |
| **8** | A retrospective belongs to a valid team; a member of that team may create or close it. | 2 | 2 | **4** | Incorrect or nonexistent team references can orphan records; allowing a non-member to act would violate the described workflow. Authentication is absent, so verify how membership is represented and enforced by the API rather than assuming it. |
| **9** | Feedback belongs to and can be retrieved for the correct retrospective. | 2 | 2 | **4** | Misassociated or missing feedback undermines the record even when create requests succeed. Verify retrieval after creation and ensure items from another retrospective are not returned. |
| **10** | Action items have valid priority and completion state and can be filtered by priority and completed status. | 2 | 2 | **4** | Incorrect filters or invalid action-item data can hide work or give teams misleading progress. Test each priority, both completion values, and representative combined filter use if supported. |
| **11** | Retrospective title, date, and status and feedback content/type/submitter are represented correctly. | 2 | 1 | **2** | Basic data validation and mapping matter, but defects here are generally more visible and recoverable than silent state-rule violations. Check required fields, enum values, malformed values, and round-trip response data after the higher-impact rules. |

### Execution order and rationale

1. **Closed-record integrity (priority 1):** start with changing and deleting feedback
   after closure. It has the highest combined risk and directly tests whether finalized
   history can be trusted.
2. **Multi-user and irreversible state (priorities 2–4):** test parallel feedback
   submissions and the feedback/close race, then test retrospective reopening and
   action-item uncompletion. Concurrent record loss or an invalid final state can corrupt
   the shared history; the one-way state rules can make that corruption difficult to
   correct.
3. **Open-workflow consistency (priorities 5–6):** test the one-open-retrospective limit
   and feedback creation rules. These defects can fragment a team's work or append
   information after closure; check same-team behavior and verify that other teams remain
   independent.
4. **Valid ownership and usable data (priorities 7–9):** test team membership,
   retrospective/team/member relationships, and feedback retrieval. They underpin normal
   usage but are less likely than post-close mutation to silently rewrite established
   history.
5. **Action filtering and field validation (priorities 10–11):** cover filter accuracy,
   enum and required-field validation, and response mapping. These affect discoverability
   and usability; run them after the state-integrity checks because their direct business
   impact is generally lower.

The ranking reflects impact as well as likelihood: high-impact irreversible or
record-corrupting behavior stays ahead of ordinary validation, even where likelihood is
similar. If execution uncovers evidence that changes a likelihood or impact estimate,
update the order and record why.

## Environment and test data

### Environments

- **Primary:** a local or dedicated QA instance of the Spring Boot application, started
  from the release candidate and configured to use H2. The API test client must reach
  this live instance over HTTP.
- **Database:** disposable H2 data, isolated from developer or production data. Establish
  whether H2 is in-memory or file-backed, how long it lives, and the supported reset
  method before running tests. Do not rely on test execution order or stale database
  contents.
- **Observability:** capture the request, response status/body, and relevant application
  logs for failures. Record build/version and environment configuration with results.
- **Concurrency check:** only if time permits, use the same QA instance to submit two
  retrospective-create requests close together for one team. This is a targeted sanity
  check of the one-open rule, not a load test.
- **Multi-user concurrency check:** use parallel HTTP requests from two or more team
  members to add distinct feedback items to the same open retrospective. Verify that every
  successful response has its own item ID, every item can be retrieved exactly once, and
  each item retains the submitted content, type, and member name. This catches lost
  updates, duplicate writes, or incorrect attribution when users act at the same time.
  Also race feedback creation against closing that retrospective. The final state must be
  closed, and no feedback may be added after closure; if feedback is accepted before the
  close takes effect, it must remain intact in the closed record. Repeat the race several
  times with fresh data. This is a targeted correctness check, not a load test.

### Seed data

Create uniquely named records per run so tests remain repeatable:

- **Team A:** two named members; use it for the primary lifecycle scenarios.
- **Team B:** one named member; use it to check that the one-open rule is scoped per
  team, not globally.
- **Invalid team cases:** a team with an empty member list and, where the API allows,
  omitted or null members; also reference a nonexistent team.
- **Retrospectives:** one open retrospective for Team A, one closed retrospective with
  existing feedback, one open retrospective for Team B, and an additional retrospective
  attempt for Team A.
- **Feedback:** at least one item of each type in an open retrospective; a mix of types in
  the closed retrospective to check immutable content and deletion behavior.
- **Action items:** items across `LOW`, `MEDIUM`, and `HIGH`; include completed and
  incomplete examples so each filter result can be checked against known expected IDs.

Keep a small data ledger of IDs and expected state. Use fresh data or a documented reset
between test cases, and verify state by retrieving records after each attempted mutation.
This avoids false passes caused by a rejected request that nevertheless changed data, or
false failures caused by leftover records.

## Known limitations to report

The brief does not define exact endpoint paths, validation error formats, date boundaries,
duplicate-name behavior, or whether team membership is enforced beyond the stated
member workflow. Confirm these against the API reference and observed behavior; do not
invent expectations. Full security testing, performance/load testing, frontend testing,
and the User Directory integration are not covered by this pass.
