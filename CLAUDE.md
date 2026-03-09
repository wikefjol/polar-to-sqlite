# polar-to-sqlite

A CLI tool following the Dogsheep pattern for saving personal training data from the Polar AccessLink API into SQLite.

## Project Overview

**Pattern**: Dogsheep `{source}-to-sqlite`
**Domain**: Personal training/fitness data
**Data Source**: Polar Flow via AccessLink API v3
**Data Store**: SQLite (local file)
**Exploration**: Datasette

## Architecture

### Core Components

1. **`polar_to_sqlite/cli.py`**: Click-based CLI with subcommands
   - `auth`: OAuth flow, saves credentials
   - `exercises`: Fetch workout/exercise data (transaction-based)
   - `sleep`: Fetch sleep data
   - `activities`: Fetch daily activity summaries
   - `recharge`: Fetch recovery/ANS data
   - `sync`: Run all data commands in sequence

2. **`polar_to_sqlite/utils.py`**: Database operations
   - `save_exercise()`, `save_sleep()`, `save_activity()`, `save_recharge()`
   - `ensure_db_shape()`: Create indexes, views, FTS
   - `parse_iso8601_duration()`: Convert PT1H15M30S → seconds

3. **`polar_to_sqlite/polar.py`**: API wrapper
   - `PolarClient`: OAuth helpers, API methods
   - Transaction support for exercises/activities
   - Rate limiting (1s between requests)

### Design Decisions

#### Store Raw API Responses
Every table includes `raw_data TEXT` column with full JSON response. Rationale:
- Future-proof: API may add fields
- Debugging aid
- Enables re-processing without re-fetching
- 30-day retention window makes data precious

#### Duration Format: Both ISO8601 and Seconds
- Store original (`duration`: `"PT1H15M30S"`)
- Store computed (`duration_seconds`: `4530`)
- Rationale: Preserves original, enables SQL math

#### Sample Data: CSV Strings (Not Rows)
- Heart rate, speed, cadence samples come as "60,62,65,68..."
- Store as TEXT, don't explode to individual rows
- Rationale: 1-hour run at 1Hz = 3600 points × N sample types = too many rows
- Parse in application layer or Datasette queries if needed

#### Incremental Sync: Transaction Pattern
- Polar API uses transactions for exercises/activities
- Transaction returns only new data, must be committed
- No custom "last synced" tracking needed
- Sleep/recharge: upsert on date (API returns last 30 days every time)

#### Auth: Manual Copy-Paste (v0.1)
- Interactive prompt for OAuth URL
- User pastes authorization code back
- Simpler than local server, no port conflicts
- Local server can be added in v0.2 without breaking changes

### Data Retention & Limitations

**CRITICAL**: Polar AccessLink API only returns data from the last **30 days**.

Implications:
- Must run `sync` at least monthly or data is lost
- Historical backfill from API is impossible
- For older data: export TCX/GPX from Polar Flow web UI manually
- This is an API limitation, not solvable in the tool

Mitigation:
- Document prominently in README
- Consider adding --export-tcx for archival in future version

## Dogsheep Ecosystem Conventions

### CLI Patterns
- `db_path` as first positional argument (consistent across ecosystem)
- `-a/--auth` option for auth file path (default: `auth.json`)
- `--all` flag for "ignore existing, fetch everything"
- `-s/--silent` for progress bar suppression

### Database Patterns
- `alter=True` everywhere (let sqlite-utils infer schema)
- `replace=True` for upserts
- Explicit foreign keys on insert when possible
- `ensure_db_shape()` at end of every command:
  1. Add foreign keys that couldn't be set during insert
  2. Index all FK columns
  3. Enable FTS on configured columns
  4. Create views

### Table Design
- Natural primary keys from API when available (e.g., `id`, `date`)
- Foreign keys defined inline during insert
- `raw_data TEXT` column for full API response
- Denormalize where it simplifies queries (e.g., computed duration_seconds)

### Testing
- JSON fixtures from real API responses (anonymized)
- In-memory databases: `sqlite_utils.Database(memory=True)`
- Test save functions directly (core logic)
- Mock API calls with `requests-mock` for CLI tests
- Validate: tables, foreign keys, data transforms, views

## File Organization

```
polar-to-sqlite/
├── polar_to_sqlite/
│   ├── __init__.py          # version = "0.1.0"
│   ├── cli.py               # Click commands
│   ├── utils.py             # DB operations + helpers
│   └── polar.py             # API wrapper
├── tests/
│   ├── fixtures/
│   │   ├── exercise.json    # Anonymized API responses
│   │   ├── sleep.json
│   │   ├── activity.json
│   │   └── recharge.json
│   ├── test_exercises.py
│   ├── test_sleep.py
│   ├── test_activities.py
│   └── test_utils.py
├── docs/
│   └── dogsheep-spike-findings.md  # Research doc
├── .gitignore
├── pyproject.toml
├── CLAUDE.md                # This file
└── README.md                # User-facing docs
```

## Development Workflow

### Phase 0: Bootstrap (Milestone: "Project Bootstrap") ✅
- [x] Rename directory, init git
- [x] Create project structure
- [x] Write pyproject.toml, CLAUDE.md, .gitignore
- [ ] Create GitHub repo
- [ ] Set up milestones and issues

### Phase 1: Auth & API (Milestone: "Authentication")
Issue #1: Implement PolarClient OAuth flow
Issue #2: Implement `polar-to-sqlite auth` command
Issue #3: Test auth with real Polar account
Issue #4: Create API response fixtures

### Phase 2: Core Data (Milestone: "Exercise Sync")
Issue #5: Implement exercise transaction logic
Issue #6: Implement `save_exercise()` with schema
Issue #7: Implement `polar-to-sqlite exercises` command
Issue #8: Write tests for exercise import
Issue #9: Verify in datasette

### Phase 3: Additional Data (Milestone: "Full Sync")
Issue #10: Implement sleep endpoint + save_sleep()
Issue #11: Implement activities transaction + save_activity()
Issue #12: Implement recharge endpoint + save_recharge()
Issue #13: Implement `polar-to-sqlite sync` command
Issue #14: Integration tests for full sync

### Phase 4: Polish (Milestone: "v0.1.0 Release")
Issue #15: Write comprehensive README
Issue #16: Create example datasette metadata.yml
Issue #17: Error handling and user-friendly messages
Issue #18: Release v0.1.0

## Table Schemas (Target)

### exercises
PK: `id`. Columns: `start_time`, `duration`, `duration_seconds`, `calories`, `distance`, `heart_rate_avg`, `heart_rate_max`, `training_load`, `sport`, `detailed_sport`, `device`, `has_route`, `running_index`, `raw_data`.

### heart_rate_zones
FK: `exercise_id`. Columns: `zone_index`, `lower_limit`, `upper_limit`, `in_zone_seconds`.

### training_load_pro
FK: `exercise_id`. Columns: `date`, `cardio_load`, `muscle_load`, `perceived_load`, interpretations.

### exercise_samples
FK: `exercise_id`. Columns: `sample_type`, `sample_type_name`, `recording_rate`, `data` (CSV string).

### sleep
PK: `date`. Columns: sleep times, `duration_seconds`, stage durations (as seconds), `sleep_score`, `sleep_charge`, interruption counts, `raw_data`.

### nightly_recharge
PK: `date`. Columns: status, `ans_charge`, `hrv_avg`, `breathing_rate`, `heart_rate_avg`, `sleep_charge`, `raw_data`.

### daily_activities
PK: `date`. Columns: `calories`, `active_calories`, `duration_seconds`, `active_steps`, `activity_distance`, `bmr`, `raw_data`.

### physical_info
Auto PK. Columns: `transaction_id`, `created`, `weight`, `height`, HR thresholds, `vo2_max`.

## Key Patterns

### Fetch/Save Separation
```python
# Fetch from API (generator)
def fetch_exercises(client, user_id, access_token):
    transaction = client.create_exercise_transaction(user_id, access_token)
    for url in transaction.list_exercises()["exercises"]:
        yield transaction.get_exercise_summary(url)

# Save to DB (transformation + insert)
def save_exercise(db, summary, samples, zones):
    exercise = transform(summary)
    db["exercises"].insert(exercise, pk="id", replace=True, alter=True)
    # ... handle nested data
```

### Progress Bars
```python
exercise_urls = transaction.list_exercises()["exercises"]
with click.progressbar(exercise_urls, label="Fetching exercises") as bar:
    for url in bar:
        # process
```

### Auth Loading with Fallback
```python
def load_config(auth_path):
    try:
        return json.load(open(auth_path))
    except FileNotFoundError:
        token = os.environ.get("POLAR_ACCESS_TOKEN")
        if not token:
            raise click.ClickException("Run 'polar-to-sqlite auth' first")
        return {"access_token": token}
```

## Polar API Specifics

### OAuth URLs
- Authorization: `https://flow.polar.com/oauth2/authorization`
- Token: `https://polarremote.com/v2/oauth2/token`
- API Base: `https://www.polaraccesslink.com/v3`

### Transaction Flow
```python
# 1. Create transaction
response = POST /v3/users/{user_id}/exercise-transactions
transaction_id = response["transaction-id"]

# 2. Get exercise URLs
response = GET /v3/users/{user_id}/exercise-transactions/{transaction_id}
urls = response["exercises"]  # max 50

# 3. Fetch each exercise
for url in urls:
    summary = GET url
    samples = GET url + "/samples"
    zones = GET url + "/heart-rate-zones"

# 4. Commit (mark as retrieved)
DELETE /v3/users/{user_id}/exercise-transactions/{transaction_id}
```

### Rate Limiting
Use `time.sleep(1)` between API requests (pattern from official examples).

### Sample Types
```python
SAMPLE_TYPE_NAMES = {
    0: "heart_rate",
    1: "speed",
    2: "cadence",
    3: "altitude",
    4: "power",
    5: "stride_length",
}
```

## Commands Reference

```bash
# First time setup
polar-to-sqlite auth --client-id YOUR_ID --client-secret YOUR_SECRET

# Sync exercises (transaction-based, only new data)
polar-to-sqlite exercises polar.db

# Sync sleep (last 30 days, upserts)
polar-to-sqlite sleep polar.db

# Sync daily activities
polar-to-sqlite activities polar.db

# Sync recovery data
polar-to-sqlite recharge polar.db

# Sync everything
polar-to-sqlite sync polar.db

# Browse with datasette
datasette polar.db

# With custom auth file
polar-to-sqlite sync polar.db -a ~/polar-auth.json

# Using env var instead
export POLAR_ACCESS_TOKEN=your_token
polar-to-sqlite sync polar.db
```

## Future Enhancements (Out of Scope for v0.1)

- Local OAuth callback server (better UX)
- TCX/GPX import command (historical backfill)
- Sample data parsing (explode CSV to rows)
- Datasette dashboard YAML config
- Physical info command (body metrics)
- Training zones analysis
- PyPI package publication

## Related Dogsheep Tools

- `strava-to-sqlite`: Similar domain (future integration point)
- `github-to-sqlite`: Reference implementation
- `healthkit-to-sqlite`: Health/fitness domain
- `swarm-to-sqlite`: Simpler pattern example

## Resources

- Research: `docs/dogsheep-spike-findings.md`
- Polar API: https://www.polar.com/accesslink-api/
- Polar Admin: https://admin.polaraccesslink.com
- sqlite-utils: https://sqlite-utils.datasette.io/
- Datasette: https://datasette.io/
- Official Example: https://github.com/polarofficial/accesslink-example-python
