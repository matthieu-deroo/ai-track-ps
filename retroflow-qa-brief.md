# RetroFlow — QA Handover Brief
**Client:** StackCraft (internal tooling team)
**Issued to:** Test Engineering team
**Version:** 1.0

---

## Background

StackCraft is a mid-sized software consultancy with 12 agile development teams. They run
retrospectives using a mix of sticky notes, Miro boards, and shared Google Docs, with no
central record of what was discussed or whether action items were ever resolved. RetroFlow
is a lightweight internal tool built to fix that.

A separate development cohort has just finished a first version of RetroFlow's backend. It
compiles, it starts, and the happy path works in a demo. Nobody has tested it properly yet.
Before it is rolled out to all 12 StackCraft teams, your team has been asked to run a QA pass
and produce a release recommendation. The StackCraft CTO will do a brief review at the end of
the day — not of code, of your findings.

You are not writing the application. You are testing it — using AI to get there faster, but
you are responsible for what you sign off on.

---

## What RetroFlow does

RetroFlow is a backend REST API. There is no frontend in scope for today.

### Teams
- A **team** has a name and a list of members (stored as names, no authentication required)
- Teams can be created and retrieved
- A team must have at least one member

### Retrospectives
- A **retrospective** belongs to a team
- A retrospective has a title, a date, and a status: `OPEN` or `CLOSED`
- Any team member can create a retrospective for their team
- A retrospective can be closed by any team member
- **Once a retrospective is closed, it cannot be reopened**
- A team can have multiple retrospectives, but **only one can be `OPEN` at a time**

### Feedback items
- A **feedback item** belongs to a retrospective
- A feedback item has content (text), a type (`WENT_WELL`, `NEEDS_IMPROVEMENT`, `ACTION_ITEM`),
  and the name of the team member who submitted it
- Feedback items can be added to an `OPEN` retrospective
- **Feedback items in a `CLOSED` retrospective cannot be modified or deleted**
- Feedback items can be retrieved per retrospective

### Action items
- An **action item** is a special kind of feedback item (type `ACTION_ITEM`) that additionally
  tracks a `priority` (`LOW`, `MEDIUM`, `HIGH`) and a boolean `completed` flag
- Action items can be marked as completed
- **Completed action items cannot be uncompleted**
- Action items must be retrievable filtered by priority and by completed status

---

## Technical details

- Java 21, Spring Boot 3.x, Spring Data JPA, in-memory H2 database
- No authentication or security layer (assume all users are trusted internal users)
- All endpoints return standard REST status codes and JSON bodies
- Input validation exists on most endpoints — do not assume it is complete or correct
- A test suite exists in the repo already — it was written by the dev cohort under time
  pressure at the end of their day. Do not assume it is trustworthy. Read it like evidence,
  not like a spec.

---

## What StackCraft's CTO said at handover

> "I don't care how thorough your test plan looks on paper. I care whether I can put this in
> front of 12 teams on Monday without it embarrassing me. Tell me what's actually broken, tell
> me what you're not sure about, and tell me what you didn't have time to check. I'd rather
> hear 'we don't know' than a clean report that turns out to be wrong."

---

## What your team is producing today

By 16:00, your team delivers:

- **A risk-based test plan** — what you tested, in what order, and why, based on where the
  business impact of a defect would be highest
- **An executable automated test suite** covering the four core business rules at minimum,
  running against the live API (not just unit tests on isolated methods)
- **A bug list** with severity, reproduction steps, and evidence for every genuine defect found
- **A test quality audit** of the pre-existing test suite you inherited — including
  AI-generated tests you write yourselves later in the day
- **A pre-release quality report** with an explicit release recommendation: BLOCKED /
  CONDITIONAL / APPROVED

Use AI for as much of this as possible — generating test plans, writing test code, running
exploratory charters, reviewing your own output. The goal is not to demonstrate that you can
write test cases by hand. The goal is a defensible answer to the CTO's actual question: can
this go in front of 12 teams on Monday?

---

## What is out of scope for today

- Frontend or UI testing — there is no frontend
- Performance/load testing beyond basic sanity checks
- Full security penetration testing (a focused review is expected in Block 3, not a full pentest)
- Testing the StackCraft User Directory integration mentioned in Block 4 — that is a separate,
  time-boxed exercise later in the day

---

## Definition of done for today

You will not achieve full coverage. The goal is a report you can defend.

At the end of the day, each team presents:

1. **A live reproduction of your most important bug** — walk through the steps, show the
   actual API response, explain the business impact if it shipped
2. **Your pre-release quality report**, including the release recommendation, and whether the
   room agrees with it
3. **The most interesting thing AI got wrong while testing** — not your best-covered feature,
   the AI testing failure that taught you the most about where you can and can't trust
   AI-generated tests

Come prepared to explain what happened, why you think it happened, and what you would check
differently on your next engagement.
