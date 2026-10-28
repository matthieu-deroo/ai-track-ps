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
| RETRO-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`<br>Headers/body: none | `2xx`; retrospective belongs to `{teamId}`, title is `QA-JP Sprint Retro One`, status is `OPEN`. |
| RETRO-02 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Sprint Retro Two"}` | `4xx` or otherwise prevented from becoming a second `OPEN` retrospective. |
| RETRO-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; no more than one item has status `OPEN`; original retrospective remains open. |
| RETRO-02 | Conditional: `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{secondRetroId}` if the POST returned an ID<br>Headers/body: none | Must not show a second `OPEN` retrospective. Record whether it was persisted in another status. |
| RETRO-03 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Sprint Retro After Close"}` | `2xx`; response contains new retrospective ID. |
| RETRO-03 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`<br>Headers/body: none | `2xx`; new retrospective is `OPEN`; prior retrospective remains `CLOSED`. |
| RETRO-04 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/99999999/retrospectives`<br>Header: `Content-Type: application/json`<br>Body: `{"title":"QA-JP Retro Invalid Team"}` | `4xx`; retrospective is not created for the nonexistent team. First confirm `99999999` is unused. |
| CLOSE-01 | `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/close`<br>Headers/body: none | `2xx`; close operation succeeds. |
| CLOSE-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`<br>Headers/body: none | `2xx`; status is `CLOSED`. |
| CLOSE-02 | Exploratory probe: `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`<br>Header: `Content-Type: application/json`<br>Body: `{"status":"OPEN"}` | Expected `4xx` because this update route is not documented. If accepted, check whether it improperly reopens the retrospective. |
| CLOSE-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`<br>Headers/body: none | `2xx`; status remains `CLOSED`. |
| CLOSE-03 | `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/close`<br>Headers/body: none | May be `2xx` (idempotent) or `4xx`; either way it must not reopen or otherwise alter the retrospective. |
| CLOSE-03 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`<br>Headers/body: none | `2xx`; status remains `CLOSED` and data is unchanged. |
| FEEDBACK-01 | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"The demo went smoothly","type":"WENT_WELL","submittedBy":"Alice"}` | `2xx`; response contains created feedback ID. |
| FEEDBACK-01 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; item appears exactly once with the submitted content, type, and submitter. |
| FEEDBACK-02a | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Clear sprint goal","type":"WENT_WELL","submittedBy":"Alice"}` | `2xx`; feedback is created. |
| FEEDBACK-02b | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Handoffs were delayed","type":"NEEDS_IMPROVEMENT","submittedBy":"Bob"}` | `2xx`; feedback is created. |
| FEEDBACK-02c | `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Header: `Content-Type: application/json`<br>Body: `{"content":"Document deployment steps","type":"ACTION_ITEM","submittedBy":"Alice"}` | `2xx`; feedback is created. |
| FEEDBACK-02 | `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`<br>Headers/body: none | `2xx`; all three submitted items appear once with matching content and type. |
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

## Running order and ID dependencies

Run TEAM-01 or TEAM-02 first and save its `teamId`. Use that ID for RETRO-01. Use the
created `retroId` for CLOSE-01, FEEDBACK-01/02/03, and the closed-feedback cases after
closing it. For RETRO-02, the first retrospective must still be open. For RETRO-03, close
the first retrospective before creating the next one. Use the same `retroId` and
`feedbackId` consistently for CLOSE-02/03 and CLOSED-FEEDBACK-02/03.
