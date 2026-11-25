# Status codes and error words

Every Arcanist service answers with these statuses, and each one always means the same thing. web and mobile decide
what to do from the status alone (log out, retry, show "can't connect", show the error), so a status used for
something else makes the apps do the wrong thing.

Each service's contract (`swagger/v2/swagger.yaml` in api, `openapi.yaml` in the others) lists what each route sends.
This file says what each status means.

## Success

| Status | Means | Example |
|---|---|---|
| 200 | Done, here's the answer. The default for reads, updates and creates. | `GET /api/v2/current-user` |
| 201 | Created something new, **only** on a route that can also answer 200 for "it already existed". | chat `POST /dm/rooms`: 201 new room, 200 your existing 1:1 with that person |
| 202 | Started, not finished: ask again (send the same request with the same key) | a token mint still running; a quest waiting for an admin |

No 204: an empty body breaks app code that reads the answer. Send `{ "success": true }` instead.

## Errors

| Status | Means | Not for | What the apps do |
|---|---|---|---|
| 400 | The request is malformed: a missing or wrong field, bad JSON | a rule saying no (422) | show the error |
| 401 | Not logged in: no token, or it's expired or revoked | "not allowed" (403) | refresh the token and retry once (`Token has expired`); log out when the session can't be refreshed |
| 403 | Logged in, but not allowed | "not logged in" (401) | show the error; `account_pending_deletion`: the locked screen |
| 404 | It doesn't exist, or the caller may not know it does (a community someone is banned from answers 404, like one that's gone) | a missing setting or relation that should exist (500) | the screen says so ("This community isn't available") or goes back |
| 409 | Conflicts with the current state: a used key, already opened | | show the error |
| 422 | Well-formed, but a rule says no: a limit, a balance, an expired invite, a wrong quest answer | a malformed request (400) | show the error |
| 429 | Too many requests: try again later, with a `Retry-After` header | a daily or count limit (422: retrying won't help) | the outbox waits and retries; elsewhere, show the error |
| 500 | Our fault: a bug or broken data | anything the person can fix | show a fixed message |
| 502 | A service we depend on failed or answered wrongly | | "Can't connect"; the outbox retries |
| 503 | A service we depend on is down: retry later | | "Can't connect"; the outbox retries |

Rules:
- **Never 401 for "not allowed".** The apps refresh the token and may log the person out.
- **Never 200 with an error inside** (`{ "success": false }` with 200). The apps show it as data: an empty list instead of "Couldn't load".
- **Never forward another service's error unchanged.** Its status means something different to the apps here (its 401 is about *our* secret, not the person's login): answer 502 with your own `{ error }`.

## Error body

Every error is one shape:

```json
{ "error": "This invite has expired and cannot be used." }
```

`error` is a sentence a person can read: the apps show it in the toast. Never an exception's own text
("undefined method '[]' for nil"): log that, and send a fixed sentence.

Some older api routes still send other shapes (`{ "message" }`, `{ "errors": [...] }`); they are being converted.

## Words the apps react to

These exact `error` values change what the apps do. Never reword them; a new one goes here first.

| Status | `error` | Sent by | The apps |
|---|---|---|---|
| 401 | `Token has expired` | api (and every service that asks api's `GET /api/v2/whoami`) | refresh the token and retry once |
| 403 | `account_pending_deletion` | the same | go to the locked screen |
| 503 | `Auth service unavailable` | the same, when api or jwt-server couldn't check the token | wait and retry; never log out |

api also sends 403 `age_verification_required` to someone who hasn't answered the birthdate step. The apps don't
react to the word: they send that person to the step from the account's own `age_verified`.

## Sockets (chat)

| Close code | Means | The apps |
|---|---|---|
| 1008 + `Authentication required` | the token is missing or expired | refresh and reconnect |
| 1008 + any other reason | refused for good (not a member, no access, the channel was deleted, or api's 403 word) | stop |
| 1013 | api couldn't check the token or the access | wait and reconnect |
| anything else (1011 our fault, a restart) | | wait and reconnect; never log out |
