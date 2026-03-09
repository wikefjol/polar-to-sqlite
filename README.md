# polar-to-sqlite

[![License](https://img.shields.io/badge/license-Apache%202.0-blue.svg)](https://github.com/filipberntsson/polar-to-sqlite/blob/main/LICENSE)
[![Python](https://img.shields.io/badge/python-3.8+-blue.svg)](https://www.python.org/downloads/)

Save personal training data from the Polar AccessLink API into a local SQLite database.

Part of the [Dogsheep](https://dogsheep.github.io/) ecosystem of tools for building personal data warehouses.

## Features

- 🏃 Fetch exercises/workouts with detailed metrics
- 😴 Import sleep data and recovery metrics
- 📊 Daily activity summaries
- 💾 Store everything in a local SQLite database
- 🔍 Explore data with [Datasette](https://datasette.io/)
- 🔄 Incremental sync (only fetch new data)

## Installation

```bash
pip install polar-to-sqlite
```

Or install from source:

```bash
git clone https://github.com/filipberntsson/polar-to-sqlite.git
cd polar-to-sqlite
pip install -e .
```

## Setup

### 1. Create Polar API Credentials

1. Go to https://admin.polaraccesslink.com
2. Create a new OAuth2 client
3. Note your `client_id` and `client_secret`
4. Set redirect URI to: `https://flow.polar.com/oauth2/authorization`

### 2. Authenticate

```bash
polar-to-sqlite auth --client-id YOUR_CLIENT_ID --client-secret YOUR_CLIENT_SECRET
```

This will:
1. Open your browser to the Polar authorization page
2. Prompt you to paste the authorization code
3. Exchange it for an access token
4. Save credentials to `auth.json`

## Usage

### Sync All Data

```bash
polar-to-sqlite sync polar.db
```

### Sync Specific Data Types

```bash
# Exercises/workouts
polar-to-sqlite exercises polar.db

# Sleep data
polar-to-sqlite sleep polar.db

# Daily activities
polar-to-sqlite activities polar.db

# Recovery metrics
polar-to-sqlite recharge polar.db
```

### Explore with Datasette

```bash
datasette polar.db
```

Then open http://localhost:8001 in your browser.

## Data Retention

**IMPORTANT**: The Polar AccessLink API only provides data from the **last 30 days**.

To avoid data loss:
- Run `polar-to-sqlite sync polar.db` at least once per month
- Consider setting up a cron job or scheduled task

**⚠️ Historical Data Warning**: If you have years of training history in Polar Flow, the API cannot backfill it. Export your full history from https://flow.polar.com **now** as TCX/GPX files (File → Export training session). Store these exports safely even if you don't import them immediately. See [Issue #14](https://github.com/wikefjol/polar-to-sqlite/issues/14) for future import command plans.

## Authentication Options

### Auth File (Default)

Credentials are saved to `auth.json` by default:

```bash
polar-to-sqlite sync polar.db  # Uses auth.json
polar-to-sqlite sync polar.db -a ~/my-auth.json  # Custom location
```

### Environment Variable

Set `POLAR_ACCESS_TOKEN` to skip the auth file:

```bash
export POLAR_ACCESS_TOKEN=your_access_token
polar-to-sqlite sync polar.db
```

## Database Schema

### Main Tables

- **exercises**: Workout summaries (sport, duration, distance, HR, training load)
- **heart_rate_zones**: Time spent in each HR zone per exercise
- **exercise_samples**: Time-series data (heart rate, speed, cadence, etc.)
- **sleep**: Sleep duration, stages, quality metrics
- **nightly_recharge**: ANS status, HRV, recovery metrics
- **daily_activities**: Steps, calories, activity summaries

All tables include a `raw_data` column with the full API response for future extensibility.

### Views

- **exercise_summary**: Denormalized workout view with computed metrics
- **sleep_summary**: Key sleep metrics in human-readable format

## Examples

### Query Recent Runs

```sql
SELECT
    start_time,
    distance / 1000.0 as distance_km,
    duration_seconds / 3600.0 as hours,
    heart_rate_avg,
    training_load
FROM exercise_summary
WHERE sport = 'RUNNING'
ORDER BY start_time DESC
LIMIT 10;
```

### Average Sleep Score

```sql
SELECT
    AVG(sleep_score) as avg_sleep_score,
    AVG(duration_seconds / 3600.0) as avg_hours
FROM sleep
WHERE date >= date('now', '-30 days');
```

## Development

### Setup

```bash
git clone https://github.com/filipberntsson/polar-to-sqlite.git
cd polar-to-sqlite
pip install -e ".[test]"
```

### Run Tests

```bash
pytest
```

### Project Structure

```
polar-to-sqlite/
├── polar_to_sqlite/
│   ├── cli.py       # Click commands
│   ├── utils.py     # Database operations
│   └── polar.py     # API wrapper
├── tests/
│   ├── fixtures/    # API response samples
│   └── test_*.py
└── pyproject.toml
```

## Related Projects

- [dogsheep](https://dogsheep.github.io/) - Personal analytics with SQLite and Datasette
- [sqlite-utils](https://sqlite-utils.datasette.io/) - CLI tool and Python library for SQLite
- [datasette](https://datasette.io/) - Explore and publish data in SQLite

## License

Apache License 2.0

## Contributing

Contributions welcome! Please open an issue or PR.

---

**Note**: This tool is not affiliated with or endorsed by Polar Electro Oy.
