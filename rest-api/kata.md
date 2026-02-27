# REST API Kata: The Music Vault

A catalog API for bands, musicians, and the instruments they're famous for. Some instruments are legendary artifacts with names and histories — like B.B. King's "Lucille", a 1980 Gibson ES-355. Others are notable only by make and model.

**Stack:** .NET (Minimal API or Controller-based — your choice), in-memory persistence only.

---

## Domain Model

### Instrument

A physical instrument. May be a famous named artifact or simply a notable make/model.

| Field  | Type    | Notes                                            |
|--------|---------|--------------------------------------------------|
| id     | uuid    |                                                  |
| name   | string? | Famous name, if any (e.g., "Lucille")            |
| type   | enum    | `guitar`, `bass`, `drums`, `keys`, `vocals`, `other` |
| make   | string? | Manufacturer (e.g., "Gibson")                    |
| model  | string? | Model name (e.g., "ES-355")                      |
| year   | int?    | Year of manufacture                              |
| notes  | string? | Provenance, history, interesting facts           |

### Musician

| Field       | Type    | Notes       |
|-------------|---------|-------------|
| id          | uuid    |             |
| name        | string  | Required    |
| born        | date?   | Birth date  |
| nationality | string? |             |

### Band

| Field   | Type    | Notes                             |
|---------|---------|-----------------------------------|
| id      | uuid    |                                   |
| name    | string  | Required                          |
| formed  | int?    | Year formed                       |
| genre   | string? |                                   |
| active  | bool    | Whether the band currently exists |

### Membership

A musician's tenure in a band. One musician can have multiple memberships in the same band (they left and rejoined).

| Field      | Type    | Notes                                    |
|------------|---------|------------------------------------------|
| musicianId | uuid    |                                          |
| bandId     | uuid    |                                          |
| role       | string? | e.g., "Lead Guitarist", "Vocalist"       |
| since      | date?   | When they joined                         |
| until      | date?   | When they left — `null` means current    |

### InstrumentAssociation

Links a musician to a specific instrument.

| Field        | Type | Notes                                      |
|--------------|------|--------------------------------------------|
| musicianId   | uuid |                                            |
| instrumentId | uuid |                                            |
| primary      | bool | Whether this is their signature instrument |

---

## Required Endpoints

### Instruments

| Method   | Path                  | Description                                              |
|----------|-----------------------|----------------------------------------------------------|
| `GET`    | `/instruments`        | List all; filter by `?type=` and `?named=true\|false`    |
| `GET`    | `/instruments/{id}`   | Get one                                                  |
| `POST`   | `/instruments`        | Create                                                   |
| `PUT`    | `/instruments/{id}`   | Full replace                                             |
| `PATCH`  | `/instruments/{id}`   | Partial update                                           |
| `DELETE` | `/instruments/{id}`   | Delete — **only if not associated with any musician**    |

### Musicians

| Method   | Path                                          | Description                                        |
|----------|-----------------------------------------------|----------------------------------------------------|
| `GET`    | `/musicians`                                  | List all; filter by `?bandId=` and `?instrumentType=` |
| `GET`    | `/musicians/{id}`                             | Get one                                            |
| `POST`   | `/musicians`                                  | Create                                             |
| `PUT`    | `/musicians/{id}`                             | Full replace                                       |
| `PATCH`  | `/musicians/{id}`                             | Partial update                                     |
| `DELETE` | `/musicians/{id}`                             | Delete — **only if not in any band**               |
| `GET`    | `/musicians/{id}/instruments`                 | List instruments associated with this musician     |
| `PUT`    | `/musicians/{id}/instruments/{instrumentId}`  | Associate an instrument; body: `{ "primary": bool }` |
| `DELETE` | `/musicians/{id}/instruments/{instrumentId}`  | Remove the association                             |
| `GET`    | `/musicians/{id}/bands`                       | List all band memberships (past + present)         |

### Bands

| Method   | Path                                    | Description                                          |
|----------|-----------------------------------------|------------------------------------------------------|
| `GET`    | `/bands`                                | List all; filter by `?active=true\|false&genre=`     |
| `GET`    | `/bands/{id}`                           | Get one                                              |
| `POST`   | `/bands`                                | Create                                               |
| `PUT`    | `/bands/{id}`                           | Full replace                                         |
| `PATCH`  | `/bands/{id}`                           | Partial update                                       |
| `DELETE` | `/bands/{id}`                           | Delete — **only if no current members**              |
| `GET`    | `/bands/{id}/members`                   | List members (past + present) with membership detail |
| `POST`   | `/bands/{id}/members`                   | Add a member; body: `{ "musicianId", "role"?, "since"? }` |
| `PATCH`  | `/bands/{id}/members/{musicianId}`      | Update membership (role, since, until)               |
| `DELETE` | `/bands/{id}/members/{musicianId}`      | End or remove a membership (see Design Decisions)    |

---

## Rules

### HTTP Conventions

- `POST` → `201 Created` with a `Location: /resource/{id}` header.
- Successful `DELETE` → `204 No Content`.
- `GET` list endpoints → `200 OK` even when the result is empty.
- Missing resource → `404 Not Found`.
- Duplicate creation/association → `409 Conflict`.
- Validation failure → `422 Unprocessable Entity`; body must name the failing fields and explain why.
- Malformed request (bad JSON, invalid UUID) → `400 Bad Request`.

### Validation

- `Musician.name` and `Band.name` are required and non-empty.
- `Instrument.type` is required and must be one of the defined enum values.
- `Instrument.year`, if provided, must be between 1000 and the current year.
- `Membership.until` must be after `Membership.since` when both are present.
- A musician may have at most one `primary` instrument at a time — setting a new primary must automatically demote the previous one.

### PATCH Semantics

Use [JSON Merge Patch (RFC 7396)](https://www.rfc-editor.org/rfc/rfc7396): an absent field is unchanged; an explicit `null` clears the field. Required fields (`name`, `type`) cannot be nulled.

---

## Design Decisions

These are intentional ambiguities. Make an explicit choice for each and be consistent.

1. **Named instrument uniqueness** — Should the API reject a second instrument with the same `name`? (There's only one "Lucille.") Or is `name` just a label with no uniqueness constraint?

2. **`DELETE /bands/{id}/members/{musicianId}`** — Hard-delete the membership record, or soft-close it by setting `until = today`? Hard-delete loses history; soft-close means you can't re-add someone who's already a current member without resolving the open record.

3. **Musician delete guard** — "Not in any band" — does this mean no *current* membership, or no membership record at all (ever)?

4. **`PUT` behavior** — When doing a full replace, what happens to sub-resources? Does `PUT /musicians/{id}` affect their instrument associations, or are those managed only through `/musicians/{id}/instruments`?

---

## Stretch Goals

Work through these in order — each one builds on the last.

1. **Pagination** — Add `?page=1&pageSize=20` to all list endpoints. Include `X-Total-Count` in response headers and `next`/`prev` links in the response body.

2. **Name search** — `GET /musicians?q=` and `GET /bands?q=` for case-insensitive substring search on `name`.

3. **Instrument lineage** — `GET /instruments/{id}/history` — list every musician ever associated with this instrument, in chronological order of their association. This is how you trace "Lucille"'s chain of custody.

4. **Band snapshot** — `GET /bands/{id}/members?at=2005-06-01` — return the membership as it stood at a specific date.

5. **API versioning** — Introduce `/v1/` prefix. Then create a `/v2/` variant that changes one response shape (e.g., flattens the membership detail into the member list). Practice the migration story.
