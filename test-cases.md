# RetroFlow API Test Case Catalogue

## Conventions and assumptions

- Replace `<base-url>` with the QA instance URL. Send `Content-Type: application/json`
  for requests with a body.
- Use unique team names containing your initials because the hosted instance is shared.
- Capture IDs from successful create responses and use them in later requests.
- The API reference does not define exact response codes, error bodies, or all required
  fields. For successful operations, expect a successful `2xx` response and verify the
  resulting resource with a GET. For invalid or forbidden operations, expect a client
  error (`4xx`) and verify that no forbidden state change occurred; record the actual
  code/body rather than assuming a specific error format.
- The documented retrospective create request accepts a title but does not show a date
  field. Do not invent a date input expectation; note the actual date behavior as an
  API-contract question.
- Business-rule references below use the handover brief:
  - **BR-1:** A team must have at least one member.
  - **BR-2:** A team can have multiple retrospectives, but only one may be `OPEN` at a time.
  - **BR-3:** A retrospective can be closed and cannot be reopened.
  - **BR-4:** Feedback may be added to an `OPEN` retrospective.
  - **BR-5:** Feedback in a `CLOSED` retrospective cannot be modified or deleted.

## Creating a team

### TEAM-01 — Create a team with one member

**Preconditions:** The QA API is available. The team name is unique and includes the
tester initials.

**Steps:**
1. `POST <base-url>/api/teams` with:
   ```json
   {"name":"QA-JP Team One","members":["Alice"]}
   ```
2. Record the returned team ID.
3. `GET <base-url>/api/teams/{teamId}`.

**Expected result:** Creation succeeds. Retrieval returns the same team name and its one
member.

**Business rule verified:** BR-1 — one member satisfies the minimum.

### TEAM-02 — Create a team with multiple members

**Preconditions:** The QA API is available. The team name is unique and includes the
tester initials.

**Steps:**
1. Create a team with a unique name and `members` containing at least two distinct names.
2. Retrieve it using `GET /api/teams/{teamId}`.

**Expected result:** Creation succeeds and retrieval preserves the full member list
without dropping or duplicating entries.

**Business rule verified:** BR-1 — a valid non-empty member list is accepted.

### TEAM-03 — Reject a team with an empty member list

**Preconditions:** The QA API is available.

**Steps:**
1. `POST /api/teams` with a unique name and `"members":[]`.
2. If the response includes an ID despite an error, attempt to retrieve that ID.

**Expected result:** The request is rejected with a client error. No team with the
submitted name is created or retrievable.

**Business rule verified:** BR-1 — a team cannot have zero members.

### TEAM-04 — Reject a missing or null member list

**Preconditions:** The QA API is available.

**Steps:**
1. Submit one create request with the `members` property omitted.
2. Submit another create request with `"members":null`.

**Expected result:** Each invalid request is rejected, and neither creates a team. Record
the behavior separately for omitted and null values.

**Business rule verified:** BR-1 — the team must have at least one member.

## Creating a retrospective

### RETRO-01 — Create the first retrospective for a team

**Preconditions:** A valid team with at least one member exists and has no open
retrospective.

**Steps:**
1. `POST /api/teams/{teamId}/retrospectives` with
   `{"title":"QA-JP Sprint Retro One"}`.
2. Record the returned retrospective ID.
3. Retrieve it with `GET /api/retrospectives/{retrospectiveId}`.

**Expected result:** Creation succeeds. The retrospective belongs to the selected team,
has the submitted title, and is `OPEN`.

**Business rule verified:** BR-2 — the first retrospective can be open.

### RETRO-02 — Reject a second open retrospective for the same team

**Preconditions:** A team has one `OPEN` retrospective.

**Steps:**
1. Attempt to create another retrospective for that same team with a different title.
2. Retrieve the team's retrospectives using `GET /api/teams/{teamId}/retrospectives`.

**Expected result:** The request is rejected, or the API otherwise prevents a second
`OPEN` record. Retrieval confirms that no more than one retrospective for the team is
open and that the original remains unchanged.

**Business rule verified:** BR-2 — only one retrospective per team may be open.

### RETRO-03 — Allow another retrospective after the first is closed

**Preconditions:** A team has a `CLOSED` retrospective and no `OPEN` retrospective.

**Steps:**
1. Create another retrospective for the same team.
2. Retrieve the new retrospective and the team's retrospective list.

**Expected result:** Creation succeeds. The team has multiple retrospectives, but only
the new one is `OPEN`.

**Business rule verified:** BR-2 — multiple retrospectives are allowed, with at most one
open at a time.

### RETRO-04 — Reject creation for a nonexistent team

**Preconditions:** A team ID known not to exist is available.

**Steps:**
1. `POST /api/teams/{nonexistentTeamId}/retrospectives` with a valid title.

**Expected result:** The request is rejected with a client error and no retrospective is
created for an invalid team.

**Business rule verified:** Retrospectives belong to a valid team (relationship
integrity supporting BR-2).

## Closing a retrospective

### CLOSE-01 — Close an open retrospective

**Preconditions:** An `OPEN` retrospective exists.

**Steps:**
1. `PUT /api/retrospectives/{retrospectiveId}/close` with no request body.
2. Retrieve the retrospective with `GET /api/retrospectives/{retrospectiveId}`.

**Expected result:** The close request succeeds and retrieval shows status `CLOSED`.

**Business rule verified:** BR-3 — an open retrospective can be closed.

### CLOSE-02 — Probe that a closed retrospective cannot be reopened

**Preconditions:** CLOSE-01 has been completed and the retrospective is `CLOSED`.

**Steps:**
1. Probe `PUT /api/retrospectives/{retrospectiveId}` with
   `{"status":"OPEN"}`. This route is not listed in the handed-over API reference; this
   is an exploratory probe, not a documented endpoint.
2. Retrieve the retrospective with `GET /api/retrospectives/{retrospectiveId}`.

**Expected result:** The probe is rejected (for example, route/method not found or a
client error), and retrieval still shows `CLOSED`. If the service accepts the
request and changes status to `OPEN`, record a defect against BR-3.

**Business rule verified:** BR-3 — a closed retrospective cannot be reopened.

### CLOSE-03 — Close an already closed retrospective again

**Preconditions:** A retrospective is `CLOSED`.

**Steps:**
1. Repeat `PUT /api/retrospectives/{retrospectiveId}/close`.
2. Retrieve the retrospective.

**Expected result:** The request may be rejected or handled idempotently; the API
reference does not specify which. In either case, the retrospective remains `CLOSED` and
its data is unchanged.

**Business rule verified:** BR-3 — repeated close operations must not reopen or alter a
closed retrospective.

## Adding feedback to an open retrospective

### FEEDBACK-01 — Add valid feedback to an open retrospective

**Preconditions:** An `OPEN` retrospective exists. The submitted-by name is a member of
its team.

**Steps:**
1. `POST /api/retrospectives/{retrospectiveId}/feedback-items` with:
   ```json
   {
     "content":"The demo went smoothly",
     "type":"WENT_WELL",
     "submittedBy":"Alice"
   }
   ```
2. Record the feedback ID.
3. Retrieve items with `GET /api/retrospectives/{retrospectiveId}/feedback-items`.

**Expected result:** Creation succeeds. The item appears exactly once with the submitted
content, type, and submitter.

**Business rule verified:** BR-4 — feedback can be added while a retrospective is open.

### FEEDBACK-02 — Add each supported feedback type to an open retrospective

**Preconditions:** An `OPEN` retrospective exists.

**Steps:**
1. Submit valid feedback with type `WENT_WELL`.
2. Submit valid feedback with type `NEEDS_IMPROVEMENT`.
3. Submit valid feedback with type `ACTION_ITEM`.
4. Retrieve the retrospective's feedback items.

**Expected result:** All three requests succeed, and retrieval returns each item once
with its correct type and fields.

**Business rule verified:** BR-4 — valid feedback types can be added to an open
retrospective.

### FEEDBACK-03 — Reject invalid feedback input on an open retrospective

**Preconditions:** An `OPEN` retrospective exists.

**Steps:**
1. Submit feedback with an unsupported type.
2. Submit feedback with empty content.
3. Retrieve the feedback list after each request.

**Expected result:** Invalid requests are rejected and do not add items. The behavior
observed for each invalid field is recorded.

**Business rule verified:** BR-4 — only valid feedback can be added to an open
retrospective; validation expectations beyond the documented types/content are also
being explored.

## Adding, modifying, or deleting feedback in a closed retrospective

### CLOSED-FEEDBACK-01 — Reject adding feedback after closure

**Preconditions:** A retrospective is `CLOSED`; its feedback list and item IDs have been
recorded.

**Steps:**
1. Attempt to `POST /api/retrospectives/{retrospectiveId}/feedback-items` with otherwise
   valid feedback.
2. Retrieve the retrospective and its feedback items.

**Expected result:** The request is rejected and the feedback list remains unchanged.

**Business rule verified:** BR-4 — feedback may be added only while the retrospective is
open; BR-5 — closed feedback records remain protected.

### CLOSED-FEEDBACK-02 — Reject modifying feedback after closure

**Preconditions:** A closed retrospective contains a feedback item with known original
content. Record its ID and content before testing.

**Steps:**
1. `PUT /api/feedback-items/{feedbackId}` with
   `{"content":"Changed after closure"}`.
2. Retrieve the retrospective's feedback items.

**Expected result:** The update is rejected. The item still exists with its original
content and all other original fields.

**Business rule verified:** BR-5 — feedback in a closed retrospective cannot be modified.

### CLOSED-FEEDBACK-03 — Reject deleting feedback after closure

**Preconditions:** A closed retrospective contains a feedback item with a known ID.
Record the full item and feedback list before testing.

**Steps:**
1. `DELETE /api/feedback-items/{feedbackId}`.
2. Retrieve the retrospective's feedback items.

**Expected result:** The delete is rejected. The item remains present exactly once with
its original data.

**Business rule verified:** BR-5 — feedback in a closed retrospective cannot be deleted.
