# RetroFlow Postman Run Sheet

Base URL: `https://ai-track-ps-test-retroflow-production.up.railway.app`

Send `Content-Type: application/json` for each request with a JSON body. Requests marked
**none** have no body. Replace `{teamId}`, `{retroId}`, and `{feedbackId}` with IDs
returned by earlier successful requests. Use unique team names if these examples have
already been run against the shared instance.

The API reference does not specify exact response codes or error bodies. Expected error
responses below are therefore given as `4xx`; record the actual status and body.

| Test case ID | Request | Expected response |
|---|---|---|
| TEAM-01 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`<br>Header: `Content-Type: application/json`<br>Body: `{"name":"QA-JP Team One","members":["Alice"]}` | `2xx`; response contains created team ID. |
| TEAM-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}`<br>Headers/body: none | `2xx`; team name is `QA-JP Team One` and members contain `Alice`. |
| TEAM-02 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`<br>Header: `Content-Type: application/json`<br>Body: `{"name":"QA-JP Team Two","members":["Alice","Bob"]}` | `2xx`; response contains created team ID. |
| TEAM-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}`<br>Headers/body: none | `2xx`; team has both `Alice` and `Bob`, each exactly once. |
| TEAM-03 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`<br>Header: `Content-Type: application/json`<br>Body: `{"name":"QA-JP Empty Team","members":[]}` | `4xx`; empty-member team is rejected. If an ID is returned, use the next row to check persistence. |
| TEAM-03 | Conditional: `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}` using any ID returned despite the error<br>Headers/body: none | Team should not exist. Record actual result if it is retrievable. |
| TEAM-04a | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`<br>Header: `Content-Type: application/json`<br>Body: `{"name":"QA-JP Omitted Members"}` | `4xx`; request is rejected and no team is created. |
| TEAM-04b | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`<br>Header: `Content-Type: application/json`<br>Body: `{"name":"QA-JP Null Members","members":null}` | `4xx`; request is rejected and no team is created. |
| RETRO-01 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Sprint Retro One"}` | `2xx`; response contains created retrospective ID and status is `OPEN` (if returned). |
| RETRO-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; list contains `{retroId}` with the supplied title and status `OPEN`. The individual retrospective GET returned `500` in the recorded run; treat unavailable state as inconclusive. |
| RETRO-02 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Sprint Retro Two"}` | `4xx` or otherwise prevented from becoming a second `OPEN` retrospective. |
| RETRO-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; no more than one item has status `OPEN`; original retrospective remains open. |
| RETRO-02 | Conditional: `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{secondRetroId}` if the POST returned an ID<br>Headers/body: none | Must not show a second `OPEN` retrospective. Record whether it was persisted in another status. |
| RETRO-05 Setup | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`<br>Header: `Content-Type: application/json`<br>Body: `{"name":"QA-JP Concurrency Team","members":["Alice","Bob"]}` | `2xx`; save returned ID as `{concurrencyTeamId}`. Ensure this team has no retrospectives. |
| RETRO-05 A/B | Send these two requests as close together as possible (use two Postman tabs or a parallel client):<br>`POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{concurrencyTeamId}/retrospectives`<br>Header: `Content-Type: application/json`<br>Body A: `{"title":"QA-JP Concurrent Retro A"}`<br>Body B: `{"title":"QA-JP Concurrent Retro B"}` | Each request may succeed or one may be rejected. Record both responses and IDs; final state must contain at most one `OPEN` retrospective. |
| RETRO-05 Verify final state | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{concurrencyTeamId}/retrospectives`<br>Headers/body: none | `2xx`; list contains no more than one `OPEN` retrospective. If status is absent, mark result inconclusive rather than assuming success. |
| RETRO-03 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Sprint Retro After Close"}` | `2xx`; response contains new retrospective ID. |
| RETRO-03 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; new retrospective is `OPEN`; prior retrospective remains `CLOSED`. |
| RETRO-04 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/99999999/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Retro Invalid Team"}` | `4xx`; retrospective is not created for the nonexistent team. First confirm `99999999` is unused. |
| CLOSE-01 | `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/close`<br>Headers/body: none | `2xx`; close operation succeeds. |
| CLOSE-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; list contains `{retroId}` with status `CLOSED`. Use the team list because the individual retrospective GET returned `500` in the recorded run. |
| CLOSE-02 | Exploratory probe: `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`<br>Header: `Content-Type: application/json`<br>Body: `{"status":"OPEN"}` | Expected `4xx` because this update route is not documented. If accepted, check whether it improperly reopens the retrospective. |
| CLOSE-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; target `{retroId}` remains `CLOSED`. The standalone GET `/api/retrospectives/{retroId}` returned `500` in the recorded run, so treat an unavailable state as inconclusive. |
| CLOSE-03 | `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/close`<br>Headers/body: none | May be `2xx` (idempotent) or `4xx`; either way it must not reopen or otherwise alter the retrospective. |
| CLOSE-03 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; target `{retroId}` remains `CLOSED` and data is unchanged. The standalone GET `/api/retrospectives/{retroId}` returned `500` in the recorded run, so do not rely on it alone. |
| FEEDBACK-01 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"The demo went smoothly","type":"WENT_WELL","submittedBy":"Alice"}` | `2xx`; response contains created feedback ID. |
| FEEDBACK-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; item appears exactly once with the submitted content, type, and submitter. |
| FEEDBACK-02a | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Clear sprint goal","type":"WENT_WELL","submittedBy":"Alice"}` | `2xx`; feedback is created. |
| FEEDBACK-02b | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Handoffs were delayed","type":"NEEDS_IMPROVEMENT","submittedBy":"Bob"}` | `2xx`; feedback is created. |
| FEEDBACK-02c | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/action-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Document deployment steps","priority":"HIGH","submittedBy":"Alice"}` | `2xx`; action item is created with `HIGH` priority. |
| FEEDBACK-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; both ordinary feedback items appear once with matching content and type. |
| FEEDBACK-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items`<br>Headers/body: none | `2xx`; created action item is present with matching content and `HIGH` priority. |
| FEEDBACK-03a | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Bad type example","type":"INVALID_TYPE","submittedBy":"Alice"}` | `4xx`; unsupported type is rejected and no item is added. |
| FEEDBACK-03a | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; list is unchanged after the invalid-type request. |
| FEEDBACK-03b | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"","type":"WENT_WELL","submittedBy":"Alice"}` | `4xx`; empty content is rejected and no item is added. |
| FEEDBACK-03b | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; list is unchanged after the empty-content request. |
| CLOSED-FEEDBACK-01 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Late feedback attempt","type":"NEEDS_IMPROVEMENT","submittedBy":"Alice"}` | `4xx`; adding feedback to a closed retrospective is rejected. |
| CLOSED-FEEDBACK-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; list is unchanged and does not contain `Late feedback attempt`. |
| CLOSED-FEEDBACK-02 | `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/feedback-items/{feedbackId}`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Changed after closure"}` | `4xx`; update of closed feedback is rejected. |
| CLOSED-FEEDBACK-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; item still exists exactly once with original content and fields. |
| CLOSED-FEEDBACK-03 | `DELETE https://ai-track-ps-test-retroflow-production.up.railway.app/api/feedback-items/{feedbackId}`<br>Headers/body: none | `4xx`; delete of closed feedback is rejected. |
| CLOSED-FEEDBACK-03 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; item still exists exactly once with original data. |
| ACTION setup | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId2}/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Action Item Rule Retro"}` | `2xx`; save ID as `{actionRetroId}`. Use a team with no open retrospective and keep this retrospective open during ACTION-01/02. |
| ACTION-01 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{actionRetroId}/action-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Document deployment steps","priority":"HIGH","submittedBy":"Alice"}` | `2xx`; save returned ID as `{actionItemId}`. Use a dedicated open retrospective that will not be closed in the feedback tests. |
| ACTION-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=false`<br>Headers/body: none | `2xx`; newly created item appears as incomplete. If it does not, record the actual default and do not assume. |
| ACTION-01 | `PATCH https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items/{actionItemId}/complete`<br>Header: `Content-Type: application/json`<br>Body: `{"completed":true}` | `2xx`; item is marked completed. |
| ACTION-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=true`<br>Headers/body: none | `2xx`; item appears in completed results with the same ID, content, and priority. |
| ACTION-02 | `PATCH https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items/{actionItemId}/complete`<br>Header: `Content-Type: application/json`<br>Body: `{"completed":false}` | `4xx`; completed action item cannot be uncompleted. |
| ACTION-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=true`<br>Headers/body: none | `2xx`; item remains present and completed with unchanged fields. |
| ACTION-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=false`<br>Headers/body: none | `2xx`; item is absent from incomplete results. |

## Running order and ID dependencies

Run TEAM-01 or TEAM-02 first and save its `teamId`. Use that ID for RETRO-01. For RETRO-02
the first retrospective must still be open; run RETRO-05 on a fresh team with two
simultaneous POST requests. Close the first retrospective before RETRO-03. Use the team
retrospective list as the state check for CLOSE-02/03; the individual retrospective GET
returned `500` in the previous run. Use an open retrospective for FEEDBACK-01/02/03 and
create a separate dedicated open retrospective for ACTION-01/02. Close a feedback-test
retrospective before CLOSED-FEEDBACK-01/02/03; confirm `{feedbackId}` belongs to that
closed retrospective before modifying or deleting it.
