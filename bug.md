# RetroFlow Bug Log

## Failed checks from the Postman run

Source: `RetroFlow QA - Core Business Rules.postman_test_run.json`
(`2026-10-28T11:00:13Z`; 26 checks passed, 9 failed). The exported run contains
response codes but not response bodies. Findings below distinguish observed failures from
issues that still need confirmation before being treated as product defects.

| Test case | Request / precondition | Expected | Actual result / evidence | Triage |
|---|---|---|---|---|
| TEAM-03 | `POST /api/teams` with `{"name":"QA-JP {{$guid}} Empty Team","members":[]}` | Reject team with no members (`4xx`). | `201 Created`. | **Candidate defect — BR-1.** Confirm by retrieving the created team and checking whether it persists with zero members. |
| TEAM-04a | `POST /api/teams` with `{"name":"QA-JP {{$guid}} Omitted Members"}` | Reject missing members (`4xx`). | `201 Created`. | **Candidate defect — BR-1.** Confirm by retrieving the created team and checking its member list. |
| TEAM-04b | `POST /api/teams` with `{"name":"QA-JP {{$guid}} Null Members","members":null}` | Reject null members (`4xx`). | `201 Created`. | **Candidate defect — BR-1.** Confirm by retrieving the created team and checking its member list. |
| RETRO-01 retrieve | `GET /api/retrospectives/4` after retrospective creation | Successful retrieval (`2xx`). | `500 Internal Server Error`. | **Product defect candidate.** A created retrospective could not be retrieved; investigate server logs and repeat with a fresh retrospective. |
| CLOSE-01 verify | `GET /api/retrospectives/4` after close request | Successful retrieval showing `CLOSED`. | `500 Internal Server Error`. | Same failing GET as RETRO-01; closure status could not be verified. Retest after investigating the retrieval failure. |
| CLOSE-02 verify | `GET /api/retrospectives/4` after reopen probe | Successful retrieval still showing `CLOSED`. | `500 Internal Server Error`. | Same failing GET; the run provides no evidence of whether the retrospective stayed closed. |
| CLOSE-03 verify | `GET /api/retrospectives/4` after repeating close | Successful retrieval showing `CLOSED`. | `500 Internal Server Error`. | Same failing GET; the run provides no evidence of the final state. |
| FEEDBACK-02c | `POST /api/retrospectives/5/feedback-items` with type `ACTION_ITEM` | Expected `2xx` for an `ACTION_ITEM` feedback item. | `400 Bad Request`. | **Test/API contract mismatch to investigate, not yet a confirmed defect.** The API reference documents action items at `POST /api/retrospectives/{id}/action-items` with a priority field. Repeat using that endpoint and a valid priority. |
| CLOSED-FEEDBACK-03 | `DELETE /api/feedback-items/3` where the test assumes the feedback belongs to a closed retrospective | Reject deletion (`4xx`) and preserve the item. | `204 No Content`. | **Candidate defect — BR-5.** Confirm item 3 belonged to the closed retrospective and retrieve its feedback list to determine whether it was deleted. |

### Follow-up needed

The run's generic assertions checked response status codes; they did not verify response
content or persisted state for these cases. Before finalizing the bug list, retrieve the
teams created by TEAM-03/04a/04b, retry retrospective GET with a fresh record while
capturing the response body and server logs, use the documented action-item endpoint for
FEEDBACK-02c, and verify the parent retrospective and remaining feedback after the
204 delete response.

## Severity guide

- **Critical:** Data loss/corruption or a core business rule failure that makes release
  unsafe.
- **High:** Major workflow or record-integrity failure with significant business impact.
- **Medium:** Important behavior is impaired, but a workaround exists.
- **Low:** Minor issue with limited impact.

## Recording further results

For each reproduction, include the request method and URL, relevant headers and body,
response status and body, and a follow-up GET or other evidence showing whether stored
state changed. If the result differs between attempts, record each outcome and the test
conditions.
