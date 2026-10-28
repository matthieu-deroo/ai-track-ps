# Business Rule Test Coverage Review

The catalogue contains **explicit test cases for three of the four rules**, but the
Postman run mostly records response codes—not whether the returned state or content
actually satisfies the rule.

| Business rule | Explicit catalogue coverage | What the case would catch | What it would not catch / actual result |
|---|---|---|---|
| A team can only have one `OPEN` retrospective at a time | **Yes — RETRO-02**; RETRO-03 also checks that a new retrospective can be created after the previous one closes. | RETRO-02 is intended to catch a sequential second creation that leaves two open retrospectives for the same team. Its expected result also says to check that the original remains open. | It does not test simultaneous creation requests, which could expose a race-condition defect. It does not verify that separate teams can each have an open retrospective. **Recorded:** second-create request returned `409`; team-list GET returned `200`. The run does not show that the list was checked for exactly one open retrospective, so the status-code evidence alone is not a full verification. |
| A closed retrospective cannot be reopened | **Yes, but weakly — CLOSE-02** probes an undocumented `PUT /api/retrospectives/{id}` with `{"status":"OPEN"}`. CLOSE-03 only repeats the close operation. | CLOSE-02 could catch reopening if that probed endpoint accepts the request and changes the state. | The probe is not a documented route, so its rejection does not prove that no supported route can reopen a retrospective. **Recorded:** probe returned `405`, but the follow-up GET returned `500`, so the final status could not be verified. The run does **not** establish that the retrospective remained closed. |
| Feedback in a closed retrospective cannot be modified or deleted | **Yes — CLOSED-FEEDBACK-02** (modify) and **CLOSED-FEEDBACK-03** (delete). CLOSED-FEEDBACK-01 separately tests adding feedback after closure. | With a known feedback item belonging to that closed retrospective, these cases can catch an API that accepts and applies an update or deletion after closure. | They may miss a wrong or stale feedback ID, an item belonging to a different retrospective, another mutation route, or a mutation that changes data despite returning an error. The follow-up GET must check the specific item’s presence and original content—not just return `200`. **Recorded:** update returned `409` and follow-up GET returned `200`, but the export does not prove the item’s content was unchanged. Delete returned `204`, which is a candidate violation; the exported GET status alone does not confirm whether the correct item was removed or whether it belonged to the closed retrospective. |
| Completed action items cannot be uncompleted | **No.** There is no catalogue test that completes an action item and then attempts to set it incomplete. | None of the current cases would directly detect an action item changing from completed back to incomplete. | FEEDBACK-02c tries to create `ACTION_ITEM` feedback, not exercise completion state. **Recorded:** that request returned `400`; it provides no evidence about whether a completed action item can be uncompleted. Add a case using the documented action-item endpoint and completion operation: create an action item, mark it complete, attempt to mark it incomplete, then retrieve/filter it to verify it remains completed. |

## Conclusion

RETRO-02 has useful initial evidence, but the reopened-state rule needs a follow-up that
can successfully retrieve the record. The closed-feedback delete result is the strongest
candidate defect in the run, pending confirmation of the feedback item's parent and stored
state. There is currently no coverage or result for the completed-action rule.
