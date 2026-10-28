# RetroFlow API reference

This is the list of endpoints of the RetroFlow backend, as the developers handed it over. Your group has its own copy of the application running at its own URL. Put that URL in front of every path below, for example `https://<your-group-url>/api/teams`.

Do realize that this is only the shape of the API: which paths exist, which method they use and what you send. What comes back (status codes, response bodies, error messages) is not in here on purpose. Finding out what the application really does is your job today.

All requests and responses use JSON. For every request with a body, send the header `Content-Type: application/json`.

## Teams

| Method | Path | What it does |
|---|---|---|
| `POST` | `/api/teams` | Create a team |
| `GET` | `/api/teams/{id}` | Get one team |

Body for creating a team:

```json
{
  "name": "Team Phoenix",
  "members": ["Alice", "Bob"]
}
```

## Retrospectives

| Method | Path | What it does |
|---|---|---|
| `POST` | `/api/teams/{teamId}/retrospectives` | Create a retrospective for a team |
| `GET` | `/api/teams/{teamId}/retrospectives` | Get all retrospectives of a team |
| `GET` | `/api/retrospectives/{id}` | Get one retrospective |
| `PUT` | `/api/retrospectives/{id}/close` | Close a retrospective (no body) |

Body for creating a retrospective:

```json
{
  "title": "Sprint 14 retro"
}
```

## Feedback items

| Method | Path | What it does |
|---|---|---|
| `POST` | `/api/retrospectives/{retrospectiveId}/feedback-items` | Add a feedback item to a retrospective |
| `GET` | `/api/retrospectives/{retrospectiveId}/feedback-items` | Get all feedback items of a retrospective |
| `PUT` | `/api/feedback-items/{id}` | Change the content of a feedback item |
| `DELETE` | `/api/feedback-items/{id}` | Delete a feedback item |

Body for adding a feedback item. The type is one of `WENT_WELL`, `NEEDS_IMPROVEMENT` or `ACTION_ITEM`:

```json
{
  "content": "The demo went smoothly",
  "type": "WENT_WELL",
  "submittedBy": "Alice"
}
```

Body for changing a feedback item:

```json
{
  "content": "The demo went really smoothly"
}
```

## Action items

| Method | Path | What it does |
|---|---|---|
| `POST` | `/api/retrospectives/{retrospectiveId}/action-items` | Add an action item to a retrospective |
| `PATCH` | `/api/action-items/{id}/complete` | Set an action item to completed or not completed |
| `GET` | `/api/action-items` | Get action items, optionally filtered |

Body for adding an action item. The priority is one of `LOW`, `MEDIUM` or `HIGH`:

```json
{
  "content": "Fix the flaky deploy script",
  "priority": "HIGH",
  "submittedBy": "Bob"
}
```

Body for setting the completed flag:

```json
{
  "completed": true
}
```

The filter on `GET /api/action-items` uses optional query parameters, and you can combine them:

```
/api/action-items?priority=HIGH
/api/action-items?completed=false
/api/action-items?priority=HIGH&completed=false
```

## Good to know

* The IDs in the paths are the numbers the application gives to things when you create them. You'll find them in the responses.
* There is no endpoint to list all teams, or to delete a team.
* The H2 console of the original application is switched off on the hosted version.
* Your group shares one instance. Give every team you create a name with your initials in it, so you can tell your data from the other pair's.
