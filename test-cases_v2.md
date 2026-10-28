# RetroFlow API Test Case Catalogue

## Conventions and example data

- **Base URL:** `https://ai-track-ps-test-retroflow-production.up.railway.app`
- Every request below uses that full URL. Send `Content-Type: application/json` on
  requests with JSON bodies.
- Replace `{teamId}`, `{retroId}`, and `{feedbackId}` with IDs returned by earlier
  successful requests. Use fresh IDs and unique team names containing your initials
  because the hosted instance is shared. The examples use `QA-JP`; replace `JP` with
  your initials.
- For successful operations, expect a `2xx` response and verify the resulting resource
  using the documented GET endpoint. For invalid or forbidden operations, expect a `4xx`
  response and verify the protected state did not change. The API reference does not
  specify exact status codes or error bodies, so record the actual response.
- Retrospective creation is documented with a title only; no date field is shown. These
  cases follow that documented request shape and record the actual date behavior as an
  API-contract observation.
- Business-rule references:
  - **BR-1:** A team must have at least one member.
  - **BR-2:** A team may have multiple retrospectives, but only one may be `OPEN` at a time.
  - **BR-3:** A retrospective can be closed and cannot be reopened.
  - **BR-4:** Feedback may be added to an `OPEN` retrospective.
  - **BR-5:** Feedback in a `CLOSED` retrospective cannot be modified or deleted.
  - **BR-6:** A completed action item cannot be uncompleted.

## Creating a team

### TEAM-01 — Create a team with one member (positive)

**Preconditions:** API is available. `QA-JP Team One` does not already exist.

**Steps:**
1. Send `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`
   with `Content-Type: application/json` and body:
   ```json
   {"name":"QA-JP Team One","members":["Alice"]}
   ```
2. Save the `id` returned in the response as `{teamId}`.
3. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}`.

**Expected result:** Create returns `2xx`; GET returns the same name and `["Alice"]`.

**Business rule verified:** BR-1 — one member satisfies the minimum.

### TEAM-02 — Create a team with multiple members (positive)

**Preconditions:** API is available. `QA-JP Team Two` does not already exist.

**Steps:**
1. Send `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`
   with body:
   ```json
   {"name":"QA-JP Team Two","members":["Alice","Bob"]}
   ```
2. Save the response `id` as `{teamId}`.
3. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}`.

**Expected result:** Create returns `2xx`; GET returns both members exactly once.

**Business rule verified:** BR-1 — a valid non-empty member list is accepted.

### TEAM-03 — Reject an empty member list (negative)

**Preconditions:** API is available. `QA-JP Empty Team` is a fresh unique name.

**Steps:**
1. Send `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`
   with body:
   ```json
   {"name":"QA-JP Empty Team","members":[]}
   ```
2. Record the status and response body. If the response includes an `id`, send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}`
   using that ID.

**Expected result:** Create returns `4xx`; no team is created. If an ID was returned,
GET must not find a persisted team.

**Business rule verified:** BR-1 — a team cannot have zero members.

### TEAM-04 — Reject an omitted or null member list (negative)

**Preconditions:** API is available. Use a different fresh team name for each request.

**Steps:**
1. Send `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams`
   with body (member property omitted):
   ```json
   {"name":"QA-JP Omitted Members"}
   ```
2. Send a second POST to the same URL with body:
   ```json
   {"name":"QA-JP Null Members","members":null}
   ```

**Expected result:** Each request returns `4xx` and creates no team. Record the two
responses separately.

**Business rule verified:** BR-1 — a team must have at least one member.

## Creating a retrospective

### RETRO-01 — Create the first retrospective for a team (positive)

**Preconditions:** A valid team exists and currently has no open retrospective. Use the
team ID created in TEAM-01 or TEAM-02.

**Steps:**
1. Send
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`
   with body:
   ```json
   {"title":"QA-JP Sprint Retro One"}
   ```
2. Save the returned retrospective `id` as `{retroId}`.
3. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`.

**Expected result:** Create returns `2xx`; GET shows the selected team, the supplied title,
and status `OPEN`.

**Business rule verified:** BR-2 — the first retrospective can be open.

### RETRO-02 — Reject a second open retrospective for the same team (negative)

**Preconditions:** `{teamId}` has one `OPEN` retrospective; save its ID as `{firstRetroId}`.

**Steps:**
1. Send
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`
   with body:
   ```json
   {"title":"QA-JP Sprint Retro Two"}
   ```
2. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`.
3. If POST returned a second retrospective ID, send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{secondRetroId}`.

**Expected result:** The second request is rejected with `4xx`, or the system otherwise
prevents the second retrospective from being `OPEN`. The team GET shows no more than one
open retrospective, and `{firstRetroId}` remains `OPEN`.

**Business rule verified:** BR-2 — only one retrospective per team may be open.

### RETRO-05 — Concurrently create retrospectives for the same team (concurrency)

**Preconditions:** Create a fresh team with at least one member. It has no
retrospectives. Prepare two different retrospective titles.

**Steps:**
1. As close together as possible, send two parallel requests to
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`.
   Use separate requests with bodies:
   ```json
   {"title":"QA-JP Concurrent Retro A"}
   ```
   ```json
   {"title":"QA-JP Concurrent Retro B"}
   ```
2. Record both responses, including any returned IDs.
3. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`.
4. Count the retrospectives with status `OPEN` in the team list. If the list omits
   status, use the individual retrospective GET as a supplemental check; that endpoint
   returned `500` in the recorded run, so treat an unavailable status as inconclusive.

**Expected result:** At most one request may create an `OPEN` retrospective. The final
team list contains no more than one `OPEN` retrospective, regardless of whether one
request is rejected or one record is created in another status.

**Business rule verified:** BR-2 — only one retrospective per team may be open, including
when creation requests overlap.

### RETRO-03 — Create another retrospective after the first is closed (positive)

**Preconditions:** `{teamId}` has at least one `CLOSED` retrospective and no `OPEN`
retrospective. Close it using CLOSE-01 if needed.

**Steps:**
1. Send
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`
   with body:
   ```json
   {"title":"QA-JP Sprint Retro After Close"}
   ```
2. Save the new `id` as `{newRetroId}`.
3. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`.

**Expected result:** Create returns `2xx`; the new retrospective is `OPEN` and prior
retrospectives remain `CLOSED`.

**Business rule verified:** BR-2 — multiple retrospectives are allowed, with at most one
open at a time.

### RETRO-04 — Reject creation for a nonexistent team (negative)

**Preconditions:** Confirm `99999999` is not an existing team ID in the test instance.

**Steps:**
1. Send
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/99999999/retrospectives`
   with body:
   ```json
   {"title":"QA-JP Retro Invalid Team"}
   ```

**Expected result:** The request returns `4xx` and creates no retrospective for the
nonexistent team.

**Business rule verified:** Retrospectives belong to a valid team (relationship
integrity supporting BR-2).

## Closing a retrospective

### CLOSE-01 — Close an open retrospective (positive)

**Preconditions:** `{retroId}` exists and is `OPEN` (for example, from RETRO-01).

**Steps:**
1. Send
   `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/close`
   with no body.
2. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`.

**Expected result:** Close returns `2xx`; GET shows status `CLOSED`.

**Business rule verified:** BR-3 — an open retrospective can be closed.

### CLOSE-02 — Probe that a closed retrospective cannot be reopened (negative)

**Preconditions:** CLOSE-01 is complete and `{retroId}` is `CLOSED`.

**Steps:**
1. Try the exploratory request
   `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}`
   with body `{"status":"OPEN"}`. This update route is not listed in the API reference;
   this is a probe, not a documented endpoint.
2. Verify state using
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`
   and confirm the target `{retroId}` remains `CLOSED`. The standalone
   `GET /api/retrospectives/{retroId}` may also be tried, but the recorded run returned
   `500` for that endpoint, so do not rely on it as the only state check until retested.

**Expected result:** The undocumented probe is rejected (such as route/method not found
or another `4xx`), and the team-retrospective list shows the target still `CLOSED`. If
the service accepts the request and changes the status to `OPEN`, record a defect against
BR-3. Rejection of this undocumented route alone does not establish that no other route
can reopen a retrospective; check any additional status-changing route discovered in the
API or implementation.

**Business rule verified:** BR-3 — a closed retrospective cannot be reopened.

### CLOSE-03 — Close an already closed retrospective again (repeat operation)

**Preconditions:** `{retroId}` exists and is `CLOSED`.

**Steps:**
1. Repeat
   `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/close`
   with no body.
2. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/teams/{teamId}/retrospectives`
   and confirm the target `{retroId}` remains `CLOSED`. The standalone
   `GET /api/retrospectives/{retroId}` may also be checked, but do not rely on it alone
   until its recorded `500` response is retested.

**Expected result:** The repeated close may be rejected or handled idempotently; the
reference does not specify which. In either case, the retrospective remains `CLOSED` and
its data is unchanged.

**Business rule verified:** BR-3 — repeated close operations must not reopen or alter a
closed retrospective.

## Adding feedback to an open retrospective

### FEEDBACK-01 — Add valid feedback to an open retrospective (positive)

**Preconditions:** `{retroId}` is `OPEN`; Alice is a member of its team.

**Steps:**
1. Send
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`
   with body:
   ```json
   {
     "content":"The demo went smoothly",
     "type":"WENT_WELL",
     "submittedBy":"Alice"
   }
   ```
2. Save the returned feedback `id` as `{feedbackId}`.
3. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`.

**Expected result:** POST returns `2xx`; GET contains the new item exactly once with the
submitted content, type, and submitter.

**Business rule verified:** BR-4 — feedback can be added while a retrospective is open.

### FEEDBACK-02 — Add feedback and an action item to an open retrospective (positive)

**Preconditions:** `{retroId}` is `OPEN`.

**Steps:**
1. POST to
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`
   with `{"content":"Clear sprint goal","type":"WENT_WELL","submittedBy":"Alice"}`.
2. POST to the same URL with
   `{"content":"Handoffs were delayed","type":"NEEDS_IMPROVEMENT","submittedBy":"Bob"}`.
3. Add an action item using the documented action-item endpoint:
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/action-items`
   with body
   `{"content":"Document deployment steps","priority":"HIGH","submittedBy":"Alice"}`.
4. GET
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`
   to retrieve the two ordinary feedback items, and
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items`
   to retrieve action items.

**Expected result:** The two feedback POST requests and the action-item POST return `2xx`.
Feedback retrieval returns the two ordinary items once; action-item retrieval includes
the created action item with priority `HIGH` and initially incomplete status (if exposed
in the response).

**Business rule verified:** BR-4 — feedback can be added to an open retrospective; the
action-item request also checks the documented action-item creation contract.

### FEEDBACK-03 — Reject invalid feedback input (negative)

**Preconditions:** `{retroId}` is `OPEN`. Take note of the current feedback count/list.

**Steps:**
1. POST to
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`
   with `{"content":"Bad type example","type":"INVALID_TYPE","submittedBy":"Alice"}`.
2. POST to the same URL with
   `{"content":"","type":"WENT_WELL","submittedBy":"Alice"}`.
3. After each POST, GET
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`.

**Expected result:** Each invalid request returns `4xx` and does not add an item. Record
the actual validation behavior for each input.

**Business rule verified:** BR-4 — feedback can be added while open; input validation
for supported types and non-empty content is also explored.

## Adding, modifying, or deleting feedback in a closed retrospective

### CLOSED-FEEDBACK-01 — Reject adding feedback after closure (negative)

**Preconditions:** `{retroId}` is `CLOSED`. Save the current feedback list for comparison.

**Steps:**
1. POST to
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`
   with valid body:
   `{"content":"Late feedback attempt","type":"NEEDS_IMPROVEMENT","submittedBy":"Alice"}`.
2. GET
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`.

**Expected result:** POST returns `4xx`; the retrieved list is unchanged and does not
contain `"Late feedback attempt"`.

**Business rule verified:** BR-4 — feedback is added only while open; BR-5 — closed
feedback records are protected.

### CLOSED-FEEDBACK-02 — Reject modifying feedback after closure (negative)

**Preconditions:** `{retroId}` is `CLOSED` and contains `{feedbackId}` with known original
content. Retrieve the feedback list and confirm the exact item ID and that its parent is
`{retroId}`. Save the full original item.

**Steps:**
1. Send
   `PUT https://ai-track-ps-test-retroflow-production.up.railway.app/api/feedback-items/{feedbackId}`
   with body:
   ```json
   {"content":"Changed after closure"}
   ```
2. GET
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`.

**Expected result:** PUT returns `4xx`; GET shows the item still exists exactly once with
its original content and fields.

**Business rule verified:** BR-5 — feedback in a closed retrospective cannot be modified.

### CLOSED-FEEDBACK-03 — Reject deleting feedback after closure (negative)

**Preconditions:** `{retroId}` is `CLOSED` and contains `{feedbackId}`. Retrieve the
retrospective's feedback list and confirm the exact item ID, original content, and
retrospective ownership before sending the delete. Save the complete original item and
list.

**Steps:**
1. Send
   `DELETE https://ai-track-ps-test-retroflow-production.up.railway.app/api/feedback-items/{feedbackId}`
   with no body.
2. GET
   `https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/feedback-items`.

**Expected result:** DELETE returns `4xx`; GET still contains the original item exactly
once with its original data.

**Business rule verified:** BR-5 — feedback in a closed retrospective cannot be deleted.

## Completing action items

### ACTION-01 — Create and complete an action item (positive)

**Preconditions:** Use a dedicated `OPEN` retrospective that is separate from the
retrospective used in the close/closed-feedback cases. The API reference does not specify
the default completion value for newly created action items; this case assumes a new item
is incomplete and records the observed initial state.

**Steps:**
1. Send
   `POST https://ai-track-ps-test-retroflow-production.up.railway.app/api/retrospectives/{retroId}/action-items`
   with body:
   ```json
   {
     "content":"Document deployment steps",
     "priority":"HIGH",
     "submittedBy":"Alice"
   }
   ```
2. Save the returned action-item `id` as `{actionItemId}`.
3. Verify it is initially incomplete using
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=false`.
4. Send
   `PATCH https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items/{actionItemId}/complete`
   with `Content-Type: application/json` and body `{"completed":true}`.
5. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=true`.

**Expected result:** The action item is created and appears in the incomplete results
before completion. After PATCH, it appears in the completed results with the same ID,
content, and priority. If a new action item is not incomplete, record that observed
default separately; do not try to force it to incomplete after it has become completed.

**Business rule verified:** BR-6 — action items can be marked completed.

### ACTION-02 — Reject uncompleting a completed action item (negative)

**Preconditions:** ACTION-01 has completed successfully. `{actionItemId}` is confirmed
to be completed. Save its content, priority, and ID.

**Steps:**
1. Send
   `PATCH https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items/{actionItemId}/complete`
   with `Content-Type: application/json` and body `{"completed":false}`.
2. Record the status and response body.
3. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=true`.
4. Send
   `GET https://ai-track-ps-test-retroflow-production.up.railway.app/api/action-items?completed=false`.

**Expected result:** The attempt to uncomplete is rejected with `4xx`, and the same
action item remains in the `completed=true` results with unchanged fields. It must not
appear in `completed=false` results.

**Business rule verified:** BR-6 — completed action items cannot be uncompleted.
