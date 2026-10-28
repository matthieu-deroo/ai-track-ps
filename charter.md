# Exploratory Testing Charter — RetroFlow

## Session goal
Assess whether the RetroFlow backend preserves the core retrospective business rules under normal use and under edge conditions, with a particular focus on defects that would embarrass the CTO if they shipped.

This is a focused, time-boxed session. The goal is not exhaustive coverage; it is to find the highest-risk defects quickly and determine whether the service is safe to put in front of real teams.

## Time box
60–90 minutes

## Test focus
Prioritise areas where the business impact of a defect is highest:
- invalid state transitions
- data integrity issues after closure
- team/retrospective membership logic
- action-item completion and filtering
- invalid or missing input causing partial persistence
- permission-adjacent behaviour in a system with no auth layer

## Priority order

### 1) Happy-path flows to explore first

#### Team creation and retrieval
- Create a valid team with a proper name and at least one member; members are stored/saved by name
- Retrieve that team and verify record integrity, including the exact team name and saved member names
- Create a second team with a different member set
- Verify team details are isolated by team ID

Why first: team creation underpins every other flow, and a broken team model can poison retrospective and feedback behavior.

#### Retrospective lifecycle
- Create a retrospective for a team with a valid title, date, and status
- Confirm the title, date, and status are set and persisted correctly
- Verify the status is only `OPEN` or `CLOSED`
- Retrieve it and confirm status = OPEN
- Close the retrospective using any team member
- Verify it is now CLOSED and remains retrievable
- Attempt to create another retrospective on the same team while one is still OPEN
- Attempt to reopen a closed retrospective
- Check that a different team member can close a retrospective for the same team without violating the business rule

Why first: the state-machine rules are central business logic and likely to have the highest operational impact if broken.

#### Feedback item creation and retrieval
- Add feedback items to an OPEN retrospective
- Include all supported types: WENT_WELL, NEEDS_IMPROVEMENT, ACTION_ITEM
- Ensure each feedback item contains the name of the team member who submitted it
- Retrieve the feedback list for the retrospective
- Validate content, type, author, and association to the correct retrospective
- Attempt to modify or delete feedback after the retrospective is closed and confirm the operation is rejected
- Confirm that a team member cannot modify or delete feedback items created by another team member

Why first: retrospective usefulness depends on accurate capture and retrieval of feedback.

#### Action item completion and filtering
- Create an ACTION_ITEM with a priority of `LOW`, `MEDIUM`, or `HIGH` and completed = false
- Retrieve action items by priority and completion status
- Mark it completed
- Verify completed items remain retrievable and cannot be uncompleted
- Confirm that attempting to uncomplete a completed action item is rejected

Why first: action-item state is a special-case rule with explicit business constraints and a strong chance of partial implementation bugs.

### 2) Edge cases to probe immediately after happy path

#### Timing / state-boundary cases
- Create a second OPEN retrospective for the same team before closing the first one
- Close the same retrospective more than once
- Add feedback to a retrospective after it has been closed
- Try to delete or modify feedback in a closed retrospective
- Try to modify or delete someone else’s feedback item while the retrospective is still OPEN
- Mark an already-completed action item completed again
- Attempt to uncomplete a completed action item

#### Invalid or missing input
- team missing a valid name or using blank/whitespace-only names
- team created with no members or null members
- team members saved by name only, so blank names, duplicate names, or name casing inconsistencies must be treated as validation risks
- retrospective created without a title, with blank/whitespace title, or with null title
- retrospective created without a date or with an invalid date format
- retrospective created with a status not equal to `OPEN` or `CLOSED`
- null or invalid retrospective status values
- blank retrospective title/date
- null or invalid retrospective status values
- feedback item missing a submitting member name
- feedback item with blank or whitespace-only author name
- invalid feedback type values
- blank feedback content or author name
- action item priority not equal to `LOW`, `MEDIUM`, or `HIGH`
- invalid priority on action item
- malformed JSON, missing required fields, wrong content type

#### Permission-adjacent behaviour
Because there is no authentication layer, probe whether the API enforces team membership correctly:
- can a caller create a retrospective for another team without being a team member?
- can a caller close a retrospective they did not create?
- can a caller manipulate another team’s feedback or action items by using IDs?
- can a team member with a matching name but no actual team association act as if they are a valid member?
- can one team member edit or delete another team member’s feedback item by changing the author name or using a guessed ID?

This is a likely risk area in a system where identity is effectively “just a string in the request body.”

#### Concurrent / repeated-action cases
- two users create an OPEN retrospective for the same team at the same time
- two users close the same retrospective concurrently
- submit duplicate completion actions for the same action item
- add feedback while a retrospective is being closed or immediately after closure
- attempt to uncomplete the same completed action item concurrently

These are high-value checks because state validation bugs often show up under race-like conditions rather than in isolated happy-path runs.

## Underspecified areas worth probing deliberately

### 1) Team membership semantics are underdefined
The brief says a team has a list of members stored as names, with no auth required. It does not define:
- whether member names are unique within a team
- whether names are case-insensitive
- whether duplicates are allowed
- whether empty strings or whitespace-only entries are valid
- whether team membership is enforced at the API layer or only by convention
- whether the member identity is simply the saved name string rather than a separate member record

These are likely to cause inconsistency in real usage and should be tested aggressively.

### 2) “Any team member” is broad and under-specified
The brief says any team member can create or close a retrospective. In a system with no auth layer, this means the real enforcement is likely to depend on request data rather than authenticated identity.

Probe for:
- arbitrary team-member names accepted without validation
- ability to act on other teams via guessed IDs
- inconsistent handling of member names across endpoints
- other members modifying feedback authored by a different team member

### 3) Closed-state rules are clear but may be only partially enforced
The key rules are explicit:
- closed retrospectives cannot be reopened
- feedback in closed retrospectives cannot be modified or deleted
- completed action items cannot be uncompleted
- retrospective status must remain constrained to `OPEN` or `CLOSED`

But the brief does not define whether related objects must be completed before closure, or whether any workarounds exist that preserve the closed status while still permitting edits via alternate routes.

### 4) Error handling contract is not fully described
The brief says standard REST status codes and JSON bodies are expected, but not how business-rule violations are represented.

Probe whether the API consistently returns:
- 400 for invalid payloads
- 404 for missing resources
- 409 for conflicting states
- clear validation messages instead of generic failures

### 5) This is a state-machine-heavy domain with little explicit spec detail
Critical transitions are implied rather than exhaustively specified:
- when a team can be created with only one member
- whether a retrospective can be created if another team exists with the same title
- how action items behave when the retrospective is closed
- whether the closed retrospective remains queryable for historical reporting
- whether a retrospective must have a non-empty title and valid date before being persisted
- whether `status` is restricted to only `OPEN` and `CLOSED`

These are exactly the places where hidden defects often live.

## Test strategy for the session

### Phase 1: smoke and invariant checks (0–15 min)
Start with a minimal valid team, one retrospective, and one action item. Validate the core assumptions before doing anything ambitious.

### Phase 2: core business-rules pass (15–45 min)
Exercise the primary happy paths and the explicit business rules. This is where the API should either behave predictably or reveal obvious state bugs.

### Phase 3: adversarial edge probing (45–75 min)
Focus on invalid input, boundary states, inconsistent closures, and team-membership logic. This is where the “looks fine in demo” defects tend to surface.

### Phase 4: risk triage (75–90 min)
Capture only the most valuable issues: those that would create incorrect retrospective data, allow invalid state transitions, or expose the wrong team’s data.

## Exit criteria for the charter
The session is successful if it answers these three questions:
- Does the backend maintain the state model correctly for teams, retrospectives, feedback, and action items?
- Can invalid or missing data create partial or contradictory state?
- Is the API safe enough to expose to real users without auth and with team-member logic based on names alone?

If the answer to any of these is “not confidently,” the tool is not ready for a broad team roll-out without more work.

## Most important defects to look for
1. Multiple OPEN retrospectives for same team
2. Reopening or re-closing a closed retrospective
3. Editing/deleting feedback after retrospective closure
4. Uncompleting completed action items
5. Manipulating data for other teams through name or ID misuse
6. Invalid payloads persisting partial state
7. Duplicate member / membership logic undermining team isolation

These are the rules most likely to break real-world trust if left unfixed.
