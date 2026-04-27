---
name: pro1-cli
description: CLI tool for accessing PRO1 workout tracking data via Firebase auth + Hasura GraphQL
version: 1.3.0
author: Hermes Agent
license: MIT
dependencies: []
metadata:
  hermes:
    tags: [workout, fitness, CLI, GraphQL, Firebase, Hasura]
    related_skills: []
  credentials:
    file: /root/.hermes/credentials/pro1.env
    vars: [FIREBASE_API_KEY, FIREBASE_AUTH_URL, HASURA_ENDPOINT, PRO1_EMAIL, PRO1_PASSWORD]

---

# PRO1 CLI Skill

## Credentials

All sensitive credentials are stored in **`/root/.hermes/credentials/pro1.env`** (git-ignored).
**Do not hardcode any secrets in this skill file.**

Load credentials in Python:
```python
import os
from pathlib import Path

creds = {}
env_file = Path('/root/.hermes/credentials/pro1.env')
for line in env_file.read_text().strip().split('\n'):
    if '=' in line and not line.startswith('#'):
        k, v = line.split('=', 1)
        creds[k] = v.strip()

API_KEY = creds['FIREBASE_API_KEY']
FIREBASE_AUTH_URL = creds['FIREBASE_AUTH_URL']
HASURA_ENDPOINT = creds['HASURA_ENDPOINT']
EMAIL = creds['PRO1_EMAIL']
PASSWORD = creds['PRO1_PASSWORD']
```

## Context

- **Repo**: `blablanumerodeux/pro1` (private, auth via `gh` CLI)
- **Purpose**: CLI tool for accessing PRO1 workout tracking data
- **Auth**: Firebase email/password → JWT → Hasura GraphQL
- **Repo clone**: `gh repo clone blablanumerodeux/pro1 /tmp/pro1`
- **Install**: `cd /tmp/pro1 && uv pip install -e .`
- **CLI entry**: `python3 -c "from pro1_cli.cli import cli; cli()"` or install with `-e .` then `pro1`

## Quick Start — Today's WOD

```python
import requests
from pathlib import Path

# Load credentials
creds = {}
for line in Path('/root/.hermes/credentials/pro1.env').read_text().strip().split('\n'):
    if '=' in line and not line.startswith('#'):
        k, v = line.split('=', 1)
        creds[k] = v.strip()

resp = requests.post(
    f"{creds['FIREBASE_AUTH_URL']}?key={creds['FIREBASE_API_KEY']}",
    json={"email": creds['PRO1_EMAIL'], "password": creds['PRO1_PASSWORD'], "returnSecureToken": True}
)
id_token = resp.json()['idToken']

# Today's workouts
from datetime import date
today = date.today().isoformat()
query = '''
{
  training_by_date_public(where: {date: {_eq: "%s"}}) {
    id date board_id
    training_by_week_public { day week year content }
  }
}
''' % today
r = requests.post(HASURA_ENDPOINT, headers={'Authorization': f'Bearer {id_token}'}, json={'query': query})
for w in r.json()['data']['training_by_date_public']:
    print(f"Board {w['board_id']}: {w['training_by_week_public']['content'][:200]}")
```

## Authentication

```python
import requests
from pathlib import Path

creds = {}
for line in Path('/root/.hermes/credentials/pro1.env').read_text().strip().split('\n'):
    if '=' in line and not line.startswith('#'):
        k, v = line.split('=', 1)
        creds[k] = v.strip()

resp = requests.post(
    f"{creds['FIREBASE_AUTH_URL']}?key={creds['FIREBASE_API_KEY']}",
    json={"email": creds['PRO1_EMAIL'], "password": creds['PRO1_PASSWORD'], "returnSecureToken": True}
)
id_token = resp.json()['idToken']  # use as: Authorization: Bearer {id_token}
```

## Schema Introspection

**Important**: The CLI's hardcoded GraphQL queries frequently don't match the actual API schema. Always introspect first:

```python
query = '{ __schema { types { name fields { name type { name kind } } } } }'
r = requests.post(HASURA_ENDPOINT, headers={'Authorization': f'Bearer {token}'}, json={'query': query})
```

## Known Schema vs CLI Mismatches

| CLI Query | Actual Schema Issue |
|-----------|-------------------|
| `pr(..., order_by: {date: desc})` | `date` not in `pr_order_by` |
| `session` | Table doesn't exist — use `event_public` |
| `exercice.category` | `category` relation may not exist |
| `board_exercices` (plural) | Table doesn't exist — use `board_exercice` (singular) |
| `pr.reps` | **Field does NOT exist**. `pr` only has `value` (weight) and `weight_unit`. No rep tracking in DB. |

## Verified Working Queries

### Auth + Get User
```python
import requests
from pathlib import Path

creds = {}
for line in Path('/root/.hermes/credentials/pro1.env').read_text().strip().split('\n'):
    if '=' in line and not line.startswith('#'):
        k, v = line.split('=', 1)
        creds[k] = v.strip()

resp = requests.post(
    f"{creds['FIREBASE_AUTH_URL']}?key={creds['FIREBASE_API_KEY']}",
    json={"email": creds['PRO1_EMAIL'], "password": creds['PRO1_PASSWORD'], "returnSecureToken": True}
)
id_token = resp.json()['idToken']
# Damien Lambla: user_id=3445, board_id=5 (level 5)
```

### User's board
```graphql
{ user(where: {id: {_eq: 3445}}) { id firstname lastname board { id name level } } }
```

### All boards
```graphql
{ board(limit: 50, order_by: {id: asc}) { id name level } }
```
Board IDs: 1=beginner, 2=int, 3=int+, 4=adv, 5=adv+, 6=elite, 7=pro, 9=pro, 10=C (competition)

### Exercise list (by board)
```graphql
{ board_exercice(where: {board_id: {_eq: 5}}, limit: 200) { exercice { name } } }
```
Use `board_exercice` (singular) — not plural.

### Common exercice_ids
- 4 = Back Squat
- 5 = Bench Press
- 6 = Deadlift
- 12 = Power Snatch
- 13 = Squat Snatch
- 16 = Hang Power Snatch
- 17 = Hang Squat Snatch
- 18 = Hang Power Clean
- 19 = Hang Squat Clean

### Day 2 / Week 17 (2026-04-21)

HPSn PR: 135 lbs (2026-04-18). Complex: PSn x1 + HPSn x1 + OHS x1 per set. Rest 1:30min between sets.

**Board 5** (50→75% of 135 lbs):

| Set | % | Target | Actual | Plates/side |
|-----|---|--------|--------|-------------|
| 1 | 50% | 67.5 | 70 | 10+2.5 |
| 2 | 55% | 74.25 | 75 | 15 |
| 3 | 60% | 81 | 80 | 15+2.5 |
| 4 | 65% | 87.75 | 85 | 20+2.5 |
| 5 | 70% | 94.5 | 95 | 25 |
| 6 | 75% | 101.25 | 100 | 25+2.5 |

**Board 6** (50→80% of 135 lbs):

| Set | % | Target | Actual | Plates/side |
|-----|---|--------|--------|-------------|
| 1 | 50% | 67.5 | 70 | 10+2.5 |
| 2 | 55% | 74.25 | 75 | 15 |
| 3 | 60% | 81 | 80 | 15+2.5 |
| 4 | 65% | 87.75 | 85 | 20+2.5 |
| 5 | 70% | 94.5 | 95 | 25 |
| 6 | 75% | 101.25 | 100 | 25+2.5 |
| 7 | 80% | 108 | 105 | 25+5+2.5 |

Note: "Actual" = closest achievable weight with standard plates. Available: 45, 35, 25, 15, 10, 5, 2.5 lbs. Bar = 45 lbs.

### Find exercise ID by name (board_exercice join)
```graphql
{ board_exercice(where: {board_id: {_eq: 5}, exercice: {name: {_ilike: "%hang power%"}}}, limit: 10) { exercice { id name } } }
```
Note: `training_exercice_public` does NOT exist in schema — use `board_exercice` with board_id filter instead.

### Save PR (insert)
```graphql
mutation { insert_pr_one(object: {user_id: 3445, exercice_id: 6, value: 345, weight_unit: "lbs"}) { id value } }
```
- `pr` table: only `value` (weight), `weight_unit`, `user_id`, `exercice_id` — **NO reps field**
- Delete: `mutation { delete_pr_by_pk(id: PR_ID) { id } }`

### All user's PRs
```graphql
{ pr(where: {user_id: {_eq: 3445}}, order_by: {exercice_id: asc}) { id exercice_id value weight_unit created_at } }
```

### Exercises (full library)
```graphql
query { exercice(limit: 100, order_by: {name: asc}) { id name } }
```

### Videos
```graphql
query { videos(limit: 10, order_by: {name: asc}) { id name description url duration } }
```

### Competitions
```graphql
query { competitions(limit: 10, order_by: {date: desc}) { id name date location } }
```

### Today's Workout
```graphql
query { training_by_date_public(where: {date: {_eq: "YYYY-MM-DD"}}) { id date board_id training_by_week_public { day week year content } } }
```
Returns all boards' workouts for a given date. Use board_id to filter (Damien's boards: 5, 6).

**⚠️ Content truncation bug:** When querying `training_by_date_public` with a `board_id` filter (e.g. `where: {board_id: {_eq: 5}}`), the `content` field is truncated. Workaround: query ALL boards and filter in Python, or use `board_id: {_in: [5, 6]}` without individual board queries.

### Find user by name
```graphql
{ user(where: {firstname: {_ilike: "%Damien%"}}, limit: 10) { id firstname lastname } }
```

### User's boards (multi-board)
```graphql
{ users_boards(where: {user_id: {_eq: 3445}}) { board_id board { id name level } } }
```

## WOD / Workout Requests (Damien)

When Damien asks for "WOD", "wod", or "workout" → always use PRO1 CLI via this skill (Cloudflare blocks ZenPlanner calendar). Query `training_by_date_public` for today's workout.

### Percentage-Based Lifts — Plate Table Defaults
- **5 lb minimum increments** — no 2.5 lb plates
- **±5 lbs tolerance** is acceptable
- **Combine warmup sets** at the same weight into one row
- **Bar = 45 lbs**, per side = (target - 45) / 2
- Show: set, %, target, per-side, plates/side, actual, delta
- Available plates: 45, 35, 25, 15, 10, 5 lbs
- Flag any set outside ±5 lbs tolerance

Example:
```
| Set | % | Target | Per Side | Plates/side | Actual | Δ |
|-----|---|--------|----------|-------------|--------|---|
| 1-2 | 50-60% | 152-183 | 53-69 | 45+15+10 | 180 | ~-3 ✓ |
```

Damien's working sets (update if PRs change):
- Back Squat: 305×3 @ 80%
- Bench Press: 130×3
- Deadlift: 345 lbs

## Key Schema Types

- `pr` — personal records (**weight only, no reps**)
- `exercice` — exercise library (92+ exercises in Pro board)
- `board` — training boards (levels 1-8 + C)
- `board_exercice` — join table linking `board_id` → `exercice_id` (singular!)
- `event_public` — **use instead of `session`**
- `competitions` — competitions
- `videos` — exercise videos
- `training_by_date_public` / `training_by_week_public` — workout schedules
- `user` / `user_public` — user data
- `users_boards` — user-board assignments (for multi-board users)

## Gotchas

- `pr.reps` does not exist — PR table only tracks max weight, not rep schemes
- `board_exercice` is singular — CLI wrongly uses plural `board_exercices`
- `session` table doesn't exist — use `event_public`
- Token expires — check `expires_at` and call `refresh_id_token()` if needed
- **Vision tool**: `vision_analyze` on local files fails with MiniMax (no VL model). Use `mmx vision describe --image /path/to/file.jpg` via minimax-mmx-cli instead.
- **MiniMax has NO vision/VLM**: MiniMax platform has no image understanding model. Use mmx-cli for vision tasks.
- **uv for install**: No pip in venv — use `uv pip install -e /tmp/pro1`
- **`pro1` CLI binary**: The installed `pro1` shim may not import correctly — use `python3 -c "from pro1_cli.cli import cli; cli()"` or run directly from the venv
- **`training_by_date_public`**: Works correctly — returns all board workouts for a date

## Damien's Data

- user_id: `3445`
- name: Damien Lambla
- board: 5 (Level 5 — Advanced+)
- PRs (as of 2026-04-18): Deadlift 345 lbs, Hang Squat Clean 270 lbs (est. from 50%=135), Squat Snatch 135 lbs, Hang Power Snatch 135 lbs
- Working sets: Back Squat 305×3 @ 80%, Bench Press 130×3
- PROGRESS.md: `https://github.com/blablanumerodeux/pro1/blob/main/PROGRESS.md`
