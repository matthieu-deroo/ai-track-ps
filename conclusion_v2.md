# Business Rule Coverage and Results Review

This review uses the current `test-cases.md`, the Postman run export
(`RetroFlow QA - Core Business Rules.postman_test_run.json`), and the later manual
observations in `session-notes.md`. The Postman export predates the new concurrency and
action-item cases, so those later observations are not independently evidenced by that
export. Where notes say only that a test “failed” or a request “failed,” the precise HTTP
response and persisted state are not available; those results need confirmation.

| Business rule | Explicit catalogue coverage and location | What the tests can catch | What they cannot catch / recorded result |
|---|---|---|---|
| A team can only have one `OPEN` retrospective at a time | **Yes — RETRO-02** tests sequential creation while one is already open; **RETRO-05** sends two creation requests concurrently and checks the final team list. | RETRO-02 can catch a normal sequential request that creates a second open retrospective. RETRO-05 can catch a race-condition defect that permits both overlapping requests to leave open records, provided the final response exposes each record's status. | Neither case proves behavior across every service instance or database transaction schedule. If the team list omits status, the catalogue explicitly treats the result as inconclusive; the individual GET returned `500` in the old run. **Recorded:** `session-notes.md` says the simultaneous creation attempt “failed,” interpreted as the conflicting create being rejected. The note does not include response codes or a final-state snapshot, so it is positive but incomplete evidence. The old Postman export separately shows RETRO-02's second create returned `409` and the list GET returned `200`, but the export does not record an assertion of the open-count. |
| A closed retrospective cannot be reopened | **Yes, but limited — CLOSE-02** probes an undocumented `PUT /api/retrospectives/{id}` with `{"status":"OPEN"}`; **CLOSE-03** repeats the close request but is not itself a reopen attempt. | CLOSE-02 can detect reopening through that specific route if the request is accepted and state is checked afterward. The revised case checks the team-retrospective list rather than relying only on the standalone GET. | The probe is not a documented endpoint, so its rejection does not prove that no supported or alternate route can reopen a retrospective. **Recorded:** the old Postman run returned `405` for the probe, while the standalone GET returned `500`. It therefore did not establish the post-probe status. The later session notes provide no additional reopen result. Current evidence is **inconclusive**. |
| Feedback items in a closed retrospective cannot be modified or deleted | **Yes — CLOSED-FEEDBACK-02** attempts an update and verifies original content; **CLOSED-FEEDBACK-03** attempts deletion and checks item presence; **CLOSED-FEEDBACK-01** checks adding new feedback after close. | These cases can catch direct update/delete/add endpoints that accept and persist changes against a closed retrospective. They are stronger when the precondition confirms the exact feedback ID belongs to the closed retrospective and follow-up checks compare item ID and original fields. | They do not cover undiscovered alternate mutation routes or concurrent feedback mutation racing with closure. **Recorded:** session notes say adding feedback to a closed retrospective was rejected (supportive evidence) and deleting feedback succeeded (candidate violation). The old Postman export reports `409` for modifying and `204` for deleting, but its follow-up GET only has a `200` status and does not prove which item remained or was removed. Confirm item ownership and resulting state before declaring the delete finding verified. |
| Completed action items cannot be uncompleted | **Yes — newly added ACTION-01 and ACTION-02.** ACTION-01 creates and completes an action item; ACTION-02 sends `{"completed":false}` and checks both completed and incomplete filters. | If the item identity and fields are checked in the filtered results, this catches a direct completion endpoint that lets a completed item become incomplete. | It does not cover alternate update routes or concurrent uncompletion attempts. **Recorded:** `session-notes.md` says “Checked that we cannot uncomplete an action. This test failed.” This suggests the rule may be violated, but the note does not say whether the PATCH returned success, what the resulting state was, or whether an assertion/setup failed. The earlier Postman export predates ACTION-01/02 and provides no evidence for this rule. Treat as a high-priority candidate defect and repeat ACTION-01/02, recording the response and filtered results. |

## Assessment

- **One-open rule:** some positive evidence from a rejected concurrent attempt and the
  older sequential `409`, but preserve the final-state response as evidence and verify
  that the team list contains at most one open record.
- **No reopening:** not established by the current run. The tested route was undocumented
  and the state-verification GET failed; retest using a readable state source and any
  supported status-changing operations.
- **Closed feedback immutability:** the add rejection is reported as successful behavior;
  deletion is a serious candidate defect. Verify the feedback item's parent and persisted
  state after the `204`; do not infer them from status alone.
- **No uncompletion:** current manual notes suggest a failure of the rule, but lack enough
  detail to confirm it. Repeat the new ACTION-01/ACTION-02 flow and capture the PATCH
  response plus the same action item's membership in both filtered results.

The catalogue defines test coverage, not proof that a test passed. Treat results as
confirmed only when the exact request, response, and resulting stored state are recorded.
