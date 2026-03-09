# Dogsheep Ecosystem Research Spike

Research findings for building `polar-to-sqlite` following the Dogsheep pattern.

**Date:** 2026-03-09
**Researcher:** Claude Code
**Purpose:** Deep dive into Dogsheep architecture patterns to inform polar-to-sqlite implementation

---

## Table of Contents

1. [Anatomy of a `*-to-sqlite` Tool](#1-anatomy-of-a-to-sqlite-tool)
2. [sqlite-utils as the Foundation](#2-sqlite-utils-as-the-foundation)
3. [Datasette as the Query/Exploration Layer](#3-datasette-as-the-queryexploration-layer)
4. [The Polar AccessLink API](#4-the-polar-accesslink-api)
5. [Proposed Design for polar-to-sqlite](#5-proposed-design-for-polar-to-sqlite)

---

## 1. Anatomy of a `*-to-sqlite` Tool

I analyzed three representative Dogsheep tools to extract common patterns and architectural decisions.

### 1.1 github-to-sqlite (Reference Implementation)

**Repository:** https://github.com/dogsheep/github-to-sqlite
**Status:** Most mature, actively maintained (v2.9)

#### Project Structure

```
github-to-sqlite/
├── github_to_sqlite/
│   ├── __init__.py
│   ├── cli.py           # ~650 lines, all Click commands
│   └── utils.py         # ~905 lines, API + DB logic
├── tests/
│   ├── *.json          # JSON fixtures for testing
│   └── test_*.py       # Per-command test files
├── setup.py
├── README.md
└── demo-metadata.json  # Example Datasette metadata
```

#### Packaging (setup.py)

```python
setup(
    name="github-to-sqlite",
    description="Save data from GitHub to a SQLite database",
    version=VERSION,
    packages=["github_to_sqlite"],
    entry_points="""
        [console_scripts]
        github-to-sqlite=github_to_sqlite.cli:cli
    """,
    install_requires=["sqlite-utils>=2.7.2", "requests", "PyYAML"],
    extras_require={"test": ["pytest", "requests-mock", "bs4"]},
)
```

**Key observations:**
- Uses `setup.py` (not pyproject.toml) - this is older convention
- Entry point maps to `cli.cli` function decorated with `@click.group()`
- Minimal dependencies: sqlite-utils, requests, PyYAML
- Test dependencies separated via extras_require

#### CLI Structure (cli.py)

```python
import click
import sqlite_utils

@click.group()
@click.version_option()
def cli():
    "Save data from GitHub to a SQLite database"


@cli.command()
@click.option("-a", "--auth", type=click.Path(...), default="auth.json")
def auth(auth):
    "Save authentication credentials to a JSON file"
    # Interactive prompts for token
    click.echo("Create a GitHub personal user token and paste it here:")
    personal_token = click.prompt("Personal token")
    # Save to JSON file
    open(auth, "w").write(json.dumps({"github_personal_token": personal_token}, indent=4))


@cli.command()
@click.argument("db_path", type=click.Path(...), required=True)
@click.argument("repo")
@click.option("-a", "--auth", default="auth.json")
def issues(db_path, repo, auth):
    "Save issues for a specified repository, e.g. simonw/datasette"
    db = sqlite_utils.Database(db_path)
    token = load_token(auth)
    repo_full = utils.fetch_repo(repo, token)
    utils.save_repo(db, repo_full)
    issues = utils.fetch_issues(repo, token)
    utils.save_issues(db, issues, repo_full)
    utils.ensure_db_shape(db)  # Sets up FTS, foreign keys, views


def load_token(auth):
    try:
        token = json.load(open(auth))["github_personal_token"]
    except (KeyError, FileNotFoundError):
        token = None
    if token is None:
        token = os.environ.get("GITHUB_TOKEN") or None
    return token
```

**Pattern observations:**
1. **Top-level group command** with `@click.group()` - serves as namespace
2. **Separate auth command** that writes to `auth.json` (not part of data sync)
3. **Consistent argument order:** `db_path` first (positional), then specific args, then options
4. **Auth fallback:** Check file first, then environment variable
5. **Every command calls `utils.ensure_db_shape(db)`** at the end to finalize schema
6. **DB instantiation pattern:** `db = sqlite_utils.Database(db_path)` in every command

#### Database Writing Patterns (utils.py)

**Core pattern: separate fetch and save functions**

```python
def fetch_issues(repo, token=None, issue_ids=None):
    """Generator that yields issues from API"""
    headers = make_headers(token)
    if issue_ids:
        for issue_id in issue_ids:
            url = f"https://api.github.com/repos/{repo}/issues/{issue_id}"
            response = requests.get(url, headers=headers)
            yield response.json()
    else:
        url = f"https://api.github.com/repos/{repo}/issues?state=all&filter=all"
        for issues in paginate(url, headers):
            yield from issues


def save_issues(db, issues, repo):
    """Transform and insert issues into database"""
    if "milestones" not in db.table_names():
        # Create prerequisite tables with explicit schema
        db["milestones"].create(
            {"id": int, "title": str, "description": str, "creator": int, "repo": int},
            pk="id",
            foreign_keys=(("repo", "repos", "id"), ("creator", "users", "id")),
        )

    for original in issues:
        # Clean up data: remove _url fields
        issue = {k: v for k, v in original.items() if not k.endswith("url")}

        # Add foreign key reference
        issue["repo"] = repo["id"]

        # Extract nested user object
        issue["user"] = save_user(db, issue["user"])

        # Extract nested arrays
        labels = issue.pop("labels")

        # Handle nullable foreign keys
        if issue["milestone"]:
            issue["milestone"] = save_milestone(db, issue["milestone"], repo["id"])

        # Insert with schema hints
        table = db["issues"].insert(
            issue,
            pk="id",
            foreign_keys=[
                ("user", "users", "id"),
                ("assignee", "users", "id"),
                ("milestone", "milestones", "id"),
                ("repo", "repos", "id"),
            ],
            alter=True,    # Add columns if missing
            replace=True,  # Upsert behavior
            columns={      # Explicit type hints
                "user": int,
                "title": str,
                "body": str,
            },
        )

        # Many-to-many relationship for labels
        for label in labels:
            table.m2m("labels", label, pk="id")
```

**Key sqlite-utils patterns identified:**

1. **`.insert()` method signature:**
   ```python
   table.insert(
       record,              # dict to insert
       pk="id",             # primary key column
       foreign_keys=[...],  # list of (col, table, ref_col) tuples
       alter=True,          # auto-add new columns
       replace=True,        # upsert on PK conflict
       columns={...}        # explicit type hints
   )
   ```

2. **`.upsert()` for idempotent inserts:**
   ```python
   db["users"].upsert(user_data, pk="id", alter=True)
   ```

3. **`.m2m()` for many-to-many:**
   ```python
   # Creates junction table automatically
   issues_table.m2m("labels", label_dict, pk="id")
   # Creates: issues_labels(issues_id, labels_id)
   ```

4. **`.last_pk` to get inserted ID:**
   ```python
   repo_id = db["repos"].insert(repo, pk="id", replace=True).last_pk
   ```

5. **Conditional table creation:**
   ```python
   if "table_name" not in db.table_names():
       db["table_name"].create({...}, pk="id", foreign_keys=[...])
   ```

**Schema finalization pattern:**

```python
def ensure_db_shape(db):
    """Called at end of every command to finalize schema"""
    # 1. Add foreign keys that couldn't be set during insert
    ensure_foreign_keys(db)

    # 2. Index all foreign key columns
    db.index_foreign_keys()

    # 3. Enable full-text search on configured columns
    for table, columns in FTS_CONFIG.items():
        if f"{table}_fts" not in db.table_names() and table in db.table_names():
            db[table].enable_fts(columns, create_triggers=True)

    # 4. Create views for common queries
    for view, (required_tables, sql) in VIEWS.items():
        if required_tables.issubset(set(db.table_names())):
            db.create_view(view, sql, replace=True)


# Configuration dictionaries at module level
FTS_CONFIG = {
    "issues": ["title", "body"],
    "commits": ["message"],
    "pull_requests": ["title", "body"],
}

VIEWS = {
    "recent_releases": (
        {"repos", "releases"},
        """SELECT repos.html_url as repo, releases.html_url as release,
                  releases.published_at FROM releases
           JOIN repos ON repos.id = releases.repo
           ORDER BY releases.published_at DESC"""
    ),
}
```

#### Auth Patterns

**Storage:** Plain JSON file (default: `auth.json`)

```json
{
  "github_personal_token": "ghp_xxxxxxxxxxxx"
}
```

**Loading with fallback:**
```python
def load_token(auth):
    try:
        token = json.load(open(auth))["github_personal_token"]
    except (KeyError, FileNotFoundError):
        token = None
    if token is None:
        token = os.environ.get("GITHUB_TOKEN") or None
    return token
```

**Pattern:** File first, then environment variable. Returns `None` if missing (commands will fail later with helpful error).

#### Error Handling

**API errors:**
```python
class GitHubError(Exception):
    def __init__(self, message, status_code, headers=None):
        self.message = message
        self.status_code = status_code
        self.headers = headers

def paginate(url, headers=None):
    while url:
        response = requests.get(url, headers=headers)
        if response.status_code == 204:  # No content
            return
        data = response.json()
        if isinstance(data, dict) and data.get("message"):
            raise GitHubError.from_response(response)
        # Extract next page from Link header
        url = response.links.get("next", {}).get("url")
        yield data
```

**Rate limiting:** Simple `time.sleep(1)` between requests in some commands. No sophisticated backoff.

#### Incremental Sync Pattern

**Example from commits command:**
```python
@cli.command()
@click.option("--all", is_flag=True,
              help="Load all commits (not just those that have not yet been saved)")
def commits(db_path, repos, all, auth):
    db = sqlite_utils.Database(db_path)

    def stop_when(commit):
        """Stop fetching when we see a commit already in DB"""
        try:
            db["commits"].get(commit["sha"])
            return True
        except sqlite_utils.db.NotFoundError:
            return False

    if all:
        stop_when = None

    for repo in repos:
        commits = utils.fetch_commits(repo, token, stop_when)
        utils.save_commits(db, commits, repo_id)
```

**Pattern:** Pass a callback to the fetcher that checks if record exists in DB. Stop early if found.

#### Testing Patterns

```python
# tests/test_issues.py
import pytest
import pathlib
import sqlite_utils
import json

@pytest.fixture
def issues():
    """Load fixture from JSON file"""
    return json.load(open(pathlib.Path(__file__).parent / "issues.json"))

@pytest.fixture
def db(issues):
    """Create in-memory DB with test data"""
    db = sqlite_utils.Database(memory=True)
    db["repos"].insert({"id": 1}, pk="id")
    utils.save_issues(db, issues, {"id": 1})
    return db

def test_tables(db):
    assert {"issues", "users", "labels", "repos", "issues_labels", "milestones"} == set(
        db.table_names()
    )

def test_foreign_keys(db):
    from sqlite_utils.db import ForeignKey
    assert ForeignKey(
        table="issues", column="user", other_table="users", other_column="id"
    ) in db["issues"].foreign_keys
```

**Patterns:**
- JSON fixtures in `tests/` directory
- In-memory SQLite databases via `sqlite_utils.Database(memory=True)`
- Fixtures that set up minimal schema then call actual save functions
- Tests validate schema structure (tables, FKs) and data content

---

### 1.2 healthkit-to-sqlite (Domain-Specific Pattern)

**Repository:** https://github.com/dogsheep/healthkit-to-sqlite
**Status:** v1.0.1, health/fitness domain

#### Project Structure

```
healthkit-to-sqlite/
├── healthkit_to_sqlite/
│   ├── __init__.py
│   ├── cli.py           # ~60 lines, single command
│   └── utils.py         # ~148 lines, XML parsing + DB
├── tests/
│   ├── export.xml       # Sample HealthKit export
│   └── test_healthkit_to_sqlite.py
├── setup.py
└── README.md
```

**Key difference:** Only ONE command (no subcommands), processes a file rather than hitting an API.

#### CLI Structure

```python
import click
import zipfile
import sqlite_utils
from .utils import convert_xml_to_sqlite

@click.command()  # Note: @command not @group
@click.argument("export_zip", type=click.Path(exists=True), required=True)
@click.argument("db_path", type=click.Path(...), required=True)
@click.option("-s", "--silent", is_flag=True, help="Don't show progress bar")
@click.option("--xml", is_flag=True, help="Input is XML, not a zip file")
def cli(export_zip, db_path, silent, xml):
    """Convert HealthKit export zip file into a SQLite database"""
    # Handle both .zip and raw .xml
    if xml:
        fp = open(export_zip, "r")
    else:
        zf = zipfile.ZipFile(export_zip)
        # Find export.xml inside zip
        export_xml_path = None
        for filename in zf.namelist():
            if filename.endswith(".xml"):
                firstbytes = zf.open(filename).read(1024)
                if b"<!DOCTYPE HealthData" in firstbytes:
                    export_xml_path = filename
                    break
        fp = zf.open(export_xml_path)

    db = sqlite_utils.Database(db_path)

    if silent:
        convert_xml_to_sqlite(fp, db, zipfile=zf)
    else:
        with click.progressbar(length=file_length, label="Importing from HealthKit") as bar:
            convert_xml_to_sqlite(fp, db, progress_callback=bar.update, zipfile=zf)
```

**Observations:**
- **No auth needed** - processes local file
- **Progress bar integration** via `click.progressbar` with callback
- **File format detection** logic in CLI layer
- **Single-command tools** use `@click.command()` not `@click.group()`

#### XML Streaming Pattern

```python
from xml.etree import ElementTree as ET

def find_all_tags(fp, tags, progress_callback=None):
    """Stream parse XML without loading entire file into memory"""
    parser = ET.XMLPullParser(("start", "end"))
    root = None

    while True:
        chunk = fp.read(1024 * 1024)  # Read 1MB at a time
        if not chunk:
            break

        parser.feed(chunk)

        for event, el in parser.read_events():
            if event == "start" and root is None:
                root = el
            if event == "end" and el.tag in tags:
                yield el.tag, el
            root.clear()  # Critical: free memory after processing

        if progress_callback:
            progress_callback(len(chunk))
```

**Critical pattern for large files:** Use streaming parser + `.clear()` to avoid memory bloat.

#### Batch Insert Pattern

```python
def convert_xml_to_sqlite(fp, db, progress_callback=None, zipfile=None):
    records = []

    for tag, el in find_all_tags(fp, {"Record", "Workout", "ActivitySummary"}, progress_callback):
        if tag == "Record":
            record = dict(el.attrib)
            # Flatten nested metadata
            for child in el.findall("MetadataEntry"):
                record["metadata_" + child.attrib["key"]] = child.attrib["value"]

            records.append(record)

            # Batch insert every 200 records
            if len(records) >= 200:
                write_records(records, db)
                records = []

        el.clear()  # Free memory

    # Final flush
    if records:
        write_records(records, db)


def write_records(records, db):
    """Split records by type into separate tables"""
    records_by_type = {}
    for record in records:
        # Dynamic table name from record type
        table = "r{}".format(
            record.pop("type")
            .replace("HKQuantityTypeIdentifier", "")
            .replace("HKCategoryTypeIdentifier", "")
        )
        records_by_type.setdefault(table, []).append(record)

    # Bulk insert per table
    for table, records_for_table in records_by_type.items():
        db[table].insert_all(
            records_for_table,
            alter=True,
            column_order=["startDate", "endDate", "value", "unit"],
            batch_size=50,
        )
```

**Key patterns:**
1. **Batching:** Accumulate records in memory, flush periodically
2. **Dynamic tables:** Create tables based on data values (e.g., record type)
3. **`.insert_all()`** for bulk inserts (faster than individual `.insert()`)
4. **`column_order`** parameter controls column display order in DB

#### Nested Data Pattern

```python
def workout_to_db(workout, db, zipfile=None):
    record = dict(workout.attrib)

    # Flatten metadata
    for el in workout.findall("MetadataEntry"):
        record["metadata_" + el.attrib["key"]] = el.attrib["value"]

    # Serialize arrays as JSON
    record["workout_events"] = [el.attrib for el in workout.findall("WorkoutEvent")]

    # Insert parent record, get ID
    pk = db["workouts"].insert(record, alter=True, hash_id="id").last_pk

    # Parse embedded GPS points
    points = [el.attrib for el in workout.findall("WorkoutRoute/Location")]

    # Convert string values to floats
    for point in points:
        point["workout_id"] = pk
        for key in point:
            if key not in ("date", "workout_id"):
                point[key] = float(point[key])

    # Insert child records with FK
    db["workout_points"].insert_all(
        points,
        foreign_keys=[("workout_id", "workouts")],
        batch_size=50
    )
```

**Pattern:** Parent-child relationships via auto-generated ID stored in `last_pk`.

---

### 1.3 swarm-to-sqlite (Minimal Pattern)

**Repository:** https://github.com/dogsheep/swarm-to-sqlite
**Status:** v0.3.4, simple Foursquare checkins

#### Project Structure

```
swarm-to-sqlite/
├── swarm_to_sqlite/
│   ├── __init__.py
│   ├── cli.py           # ~81 lines, single command
│   └── utils.py         # ~225 lines, API + save logic
├── tests/
│   ├── checkin.json
│   └── test_save_checkin.py
├── setup.py
└── README.md
```

**Simplest Dogsheep tool:** Single command, single data type, minimal complexity.

#### CLI Structure

```python
@click.command()
@click.argument("db_path", type=click.Path(...), required=True)
@click.option("--token", envvar="FOURSQUARE_TOKEN", help="Foursquare OAuth token")
@click.option("--load", type=click.File(), help="Load checkins from this JSON file")
@click.option("--save", type=click.File("w"), help="Save checkins to this JSON file")
@click.option("--since", type=str, callback=validate_since,
              help="Look for checkins since 1w/2d/3h ago")
@click.option("-s", "--silent", is_flag=True)
def cli(db_path, token, load, save, since, silent):
    """Save Swarm checkins to a SQLite database"""
    if token and load:
        raise click.ClickException("Provide either --load or --token")

    if not token and not load:
        # Interactive prompt if neither provided
        token = click.prompt("Please provide your Foursquare OAuth token", hide_input=True)

    if token:
        checkins = fetch_all_checkins(token, count_first=True, since_delta=since)
        checkin_count = next(checkins)  # First yield is count
    else:
        checkins = json.load(load)
        checkin_count = len(checkins)

    db = sqlite_utils.Database(db_path)

    with click.progressbar(length=checkin_count, label=f"Importing {checkin_count} checkins") as bar:
        for checkin in checkins:
            save_checkin(checkin, db)
            bar.update(1)

    ensure_foreign_keys(db)
    create_views(db)
```

**Patterns:**
- **`envvar` parameter** in `@click.option` for env var support
- **Interactive prompts** when required values missing
- **`click.File()` type** for file I/O (supports streaming)
- **Custom validation** via `callback` parameter
- **Progress bar with label** for user feedback

#### Incremental Fetch Pattern

```python
def fetch_all_checkins(token, count_first=False, since_delta=None):
    """Generator that yields all checkins, with optional count first"""
    params = {
        "oauth_token": token,
        "v": "20190101",  # API version
        "sort": "newestfirst",
        "limit": "250",
    }

    if since_delta:
        params["afterTimestamp"] = int(time.time() - since_delta)

    before_timestamp = None
    first = True

    while True:
        if before_timestamp is not None:
            params["beforeTimestamp"] = before_timestamp

        data = requests.get("https://api.foursquare.com/v2/users/self/checkins", params).json()

        if first:
            first = False
            if count_first:
                yield data["response"]["checkins"]["count"]

        items = data.get("response", {}).get("checkins", {}).get("items")
        if not items:
            break

        for item in items:
            yield item

        # Pagination via timestamp
        before_timestamp = item["createdAt"]
```

**Pattern:**
- **Generator with optional count yield** (for progress bars)
- **Timestamp-based pagination** (no "next" links)
- **`since_delta` for incremental** (only new data)

#### Complex Normalization Pattern

```python
def save_checkin(checkin, db):
    """Extract nested objects into separate tables"""
    checkin = dict(checkin)  # Copy to avoid mutating

    # Extract and save venue
    if "venue" in checkin:
        venue = checkin.pop("venue")
        categories = venue.pop("categories")
        venue.update(venue.pop("location"))  # Flatten location
        venue["latitude"] = venue.pop("lat")
        venue["longitude"] = venue.pop("lng")

        v = db["venues"].insert(venue, pk="id", alter=True, replace=True)

        # M2M: venue <-> categories
        for category in categories:
            cleanup_category(category)
            v.m2m("categories", category, pk="id", alter=True)

        checkin["venue"] = venue["id"]  # FK reference

    # Transform likes array into M2M
    users_likes = []
    for group in checkin["likes"]["groups"]:
        users_likes.extend(group["items"])
    del checkin["likes"]

    # Transform photos array into separate table
    photos = checkin.pop("photos")["items"]

    # Insert main checkin
    checkins_table = db["checkins"].insert(
        checkin,
        pk="id",
        foreign_keys=(("venue", "venues", "id"), ("source", "sources", "id")),
        alter=True,
        replace=True,
    )

    # M2M: checkin <-> users (likes)
    for user in users_likes:
        cleanup_user(user)
        checkins_table.m2m("users", user, m2m_table="likes", pk="id", alter=True)

    # Insert related photos
    for photo in photos:
        photo["created"] = datetime.datetime.utcfromtimestamp(photo["createdAt"]).isoformat()
        photo["source"] = db["sources"].lookup(photo["source"])
        photo["user"] = save_user(db, photo["user"])
        db["photos"].insert(photo, replace=True, alter=True)


def cleanup_category(category):
    """Flatten nested objects in-place"""
    category["icon_prefix"] = category["icon"]["prefix"]
    category["icon_suffix"] = category["icon"]["suffix"]
    del category["icon"]
```

**Patterns:**
- **Deep nesting extraction:** Pull out nested objects into separate tables
- **`.lookup()` method:** Upsert-like behavior for lookup tables
- **Custom M2M table names:** `m2m_table="likes"` instead of default
- **Timestamp conversion:** Unix epoch → ISO8601 strings
- **In-place cleanup helpers** for repeated transformations

#### View Creation Pattern

```python
def create_views(db):
    """Create denormalized views for common queries"""
    for name, sql in (
        (
            "venue_details",
            """SELECT
                   min(created) as first,
                   max(created) as last,
                   count(venues.id) as count,
                   group_concat(distinct categories.name) as venue_categories,
                   venues.*
               FROM venues
               JOIN checkins ON checkins.venue = venues.id
               JOIN categories_venues ON venues.id = categories_venues.venues_id
               JOIN categories ON categories.id = categories_venues.categories_id
               GROUP BY venues.id""",
        ),
        (
            "checkin_details",
            """SELECT
                   checkins.id,
                   strftime('%Y-%m-%dT%H:%M:%S', checkins.createdAt, 'unixepoch') as created,
                   venues.name as venue_name,
                   venues.latitude,
                   venues.longitude,
                   group_concat(categories.name) as venue_categories,
                   checkins.shout
               FROM checkins
               JOIN venues ON checkins.venue = venues.id
               LEFT JOIN events ON checkins.event = events.id
               JOIN categories_venues ON venues.id = categories_venues.venues_id
               JOIN categories ON categories.id = categories_venues.categories_id
               GROUP BY checkins.id
               ORDER BY checkins.createdAt DESC""",
        ),
    ):
        try:
            db.create_view(name, sql, replace=True)
        except Exception:
            pass  # Fail silently if dependencies missing
```

**Pattern:** Pre-compute common JOINs and aggregations in views for Datasette performance.

---

### Summary: Common Patterns Across Tools

| Aspect | Pattern | Notes |
|--------|---------|-------|
| **Package structure** | `{name}_to_sqlite/` with `cli.py` + `utils.py` | Separation of CLI logic from business logic |
| **Packaging** | `setup.py` with console_scripts entry point | Older tools use setup.py, newer could use pyproject.toml |
| **CLI framework** | Click with `@click.group()` or `@click.command()` | Group for multiple subcommands, command for single |
| **Database arg** | First positional argument, `db_path` | Consistent convention across ecosystem |
| **Auth storage** | JSON file (default `auth.json`) with env var fallback | Plain text, user-managed file |
| **API patterns** | Separate `fetch_*()` generators and `save_*()` functions | Separation of concerns, testability |
| **Batch processing** | Accumulate in list, flush at threshold (50-250 items) | Balance memory vs. transaction overhead |
| **Schema strategy** | `alter=True` everywhere, explicit FKs on insert | Let sqlite-utils infer types, add columns as needed |
| **Incremental sync** | Stop-when callbacks or timestamp parameters | Two approaches: check DB or use API time filters |
| **Testing** | pytest with JSON fixtures, in-memory DBs | Fast, isolated tests |
| **Finalization** | `ensure_db_shape()` at end: FKs, FTS, indexes, views | Separate schema setup from data insertion |

---

## 2. sqlite-utils as the Foundation

sqlite-utils is the core abstraction layer that makes Dogsheep tools possible. It provides a Pythonic API over SQLite.

### 2.1 Core Classes

```python
import sqlite_utils

# Create or open database
db = sqlite_utils.Database("mydata.db")  # File
db = sqlite_utils.Database(memory=True)  # In-memory
db = sqlite_utils.Database(conn)         # From existing connection

# Access tables
table = db["table_name"]
table = db.table("table_name", pk="id")  # With explicit PK
```

### 2.2 Key Methods

#### Database-level operations

```python
# Introspection
db.table_names()              # → ["table1", "table2"]
db.view_names()               # → ["view1"]
db.tables                     # → [Table("table1"), Table("table2")]
db["table"].exists()          # → True/False

# Views
db.create_view("name", "SELECT ...", replace=True)

# Foreign key indexing
db.index_foreign_keys()       # Create indexes on all FK columns

# Execute raw SQL
db.execute("CREATE INDEX ...")
db.executescript("...")
```

#### Table.insert() - Single record

```python
table = db["users"].insert(
    {"id": 1, "name": "Alice"},  # Record dict
    pk="id",                     # Primary key column
    foreign_keys=[               # Foreign key constraints
        ("org_id", "orgs", "id")
    ],
    alter=True,                  # Add columns if missing
    replace=True,                # Upsert on PK conflict
    columns={                    # Explicit type hints
        "id": int,
        "name": str,
        "active": bool,
    }
)

# Returns table instance
pk = table.last_pk               # Get inserted/updated PK
```

#### Table.insert_all() - Bulk insert

```python
db["users"].insert_all(
    [
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"},
    ],
    pk="id",
    batch_size=100,              # Commit every N records
    alter=True,
    replace=True,
)
```

**Performance:** Dramatically faster than individual inserts (100x+).

#### Table.upsert() - Idempotent insert

```python
# Insert or update if PK exists
db["users"].upsert(
    {"id": 1, "name": "Alice", "updated": "2023-01-01"},
    pk="id",
    alter=True
)

# Update only specified columns on conflict
db["users"].upsert(
    {"id": 1, "name": "Alice"},
    pk="id",
    alter=True
)
```

#### Table.m2m() - Many-to-many relationships

```python
# Automatically creates junction table
issues_table.m2m(
    "labels",                    # Other table name
    {"id": 123, "name": "bug"},  # Record to link
    pk="id",                     # PK of other table
    m2m_table="custom_name",     # Optional: override junction table name
    alter=True
)

# Creates tables:
# - labels (if missing)
# - issues_labels (junction) with (issues_id, labels_id)
```

#### Table.lookup() - Lookup table pattern

```python
# Insert if missing, return PK
source_id = db["sources"].lookup({"name": "mobile_app"})

# Equivalent to:
# SELECT id FROM sources WHERE name = 'mobile_app'
# or INSERT INTO sources (name) VALUES ('mobile_app') RETURNING id
```

#### Schema manipulation

```python
# Explicit table creation
db["commits"].create(
    {
        "sha": str,
        "message": str,
        "author_id": int,
        "date": str,
    },
    pk="sha",
    foreign_keys=[("author_id", "users", "id")],
    if_not_exists=True
)

# Add foreign key after creation
db["commits"].add_foreign_key("author_id", "users", "id")

# Add column
db["users"].add_column("email", str)

# Create index
db["commits"].create_index(["author_id", "date"])
```

#### Full-text search

```python
# Enable FTS on columns
db["articles"].enable_fts(
    ["title", "body"],
    create_triggers=True  # Keep FTS in sync with changes
)

# Creates:
# - articles_fts (virtual FTS5 table)
# - Triggers to maintain sync

# Search
results = db["articles"].search("quantum mechanics")
```

### 2.3 Schema Inference vs. Explicit Definition

**sqlite-utils infers types from Python values:**

```python
# Automatic inference
db["users"].insert({
    "id": 1,              # → INTEGER
    "name": "Alice",      # → TEXT
    "score": 95.5,        # → REAL
    "active": True,       # → INTEGER (0/1)
    "data": b"bytes",     # → BLOB
    "tags": ["a", "b"],   # → TEXT (JSON-serialized)
})
```

**Explicit types override inference:**

```python
db["users"].insert(
    {"id": "123", "count": "456"},
    columns={"id": str, "count": int}  # Force types
)
```

**Type mapping:**
- `int` → `INTEGER`
- `str` → `TEXT`
- `float` → `REAL`
- `bool` → `INTEGER`
- `bytes` → `BLOB`
- `dict`/`list` → `TEXT` (JSON)

### 2.4 Real-World Usage Patterns from Studied Code

**Pattern 1: Create tables lazily**
```python
# Don't pre-create schema; let first insert create table
if "milestones" not in db.table_names():
    db["milestones"].create({...}, pk="id")

# Or just insert with alter=True
db["table"].insert(record, pk="id", alter=True)
```

**Pattern 2: Explicit FK creation for cross-table references**
```python
# Can't set FK during insert if referenced table doesn't exist yet
# Solution: Add FKs at end in ensure_db_shape()

def ensure_foreign_keys(db):
    for table, column, ref_table, ref_column in FOREIGN_KEYS:
        if (table in db.table_names() and
            ref_table in db.table_names() and
            (table, column, ref_table, ref_column) not in db[table].foreign_keys):
            db[table].add_foreign_key(column, ref_table, ref_column)
```

**Pattern 3: Hash IDs for records without natural PK**
```python
# Generate deterministic ID from content
db["workouts"].insert(
    record,
    hash_id="id",  # Create SHA hash of record as ID
    alter=True
)
```

**Pattern 4: Column ordering for readability**
```python
db["records"].insert_all(
    records,
    column_order=["date", "type", "value", "unit"],  # These columns first
    alter=True
)
```

### 2.5 CLI Usage (for reference)

```bash
# Insert JSON data
sqlite-utils insert mydb.db table_name data.json

# Create index
sqlite-utils create-index mydb.db table_name col1 col2

# Enable FTS
sqlite-utils enable-fts mydb.db articles title body

# Query
sqlite-utils query mydb.db "SELECT * FROM articles WHERE ..."

# Schema introspection
sqlite-utils tables mydb.db --counts
sqlite-utils schema mydb.db
```

---

## 3. Datasette as the Query/Exploration Layer

Datasette is a web UI + JSON API for exploring SQLite databases. It's the "viewer" layer of the Dogsheep ecosystem.

### 3.1 Basic Usage

```bash
# Install
pip install datasette

# Launch against local DB
datasette mydata.db

# Multiple databases
datasette database1.db database2.db

# Immutable mode (read-only, enables caching)
datasette -i mydata.db

# Custom port
datasette mydata.db -p 8080

# Open browser automatically
datasette mydata.db -o
```

**Default URL:** http://localhost:8001

### 3.2 What You Get

1. **Browse tables:** Automatic UI for viewing table contents with pagination
2. **Filter/sort:** Click column headers, add filters via UI
3. **SQL editor:** Run arbitrary SQL queries with syntax highlighting
4. **Full-text search:** If FTS enabled, search box appears automatically
5. **Foreign key navigation:** Click FK values to jump to related record
6. **JSON API:** Every page has a `.json` endpoint

### 3.3 JSON API

Every table/query in Datasette has a JSON representation:

```bash
# Table data
curl http://localhost:8001/mydata/table_name.json

# With filters
curl http://localhost:8001/mydata/table_name.json?column=value

# Custom query
curl http://localhost:8001/mydata.json?sql=SELECT+*+FROM+table

# Single row
curl http://localhost:8001/mydata/table_name/primary_key.json
```

**Response format:**
```json
{
  "database": "mydata",
  "table": "table_name",
  "rows": [
    {"id": 1, "name": "Alice"},
    {"id": 2, "name": "Bob"}
  ],
  "truncated": false,
  "next": "page_token_if_more_results"
}
```

### 3.4 Metadata Configuration

Create `metadata.json` to customize display:

```json
{
  "title": "My Personal Data",
  "description": "Aggregated data from various services",
  "databases": {
    "polar": {
      "tables": {
        "exercises": {
          "label_column": "start_time",
          "sort_desc": "start_time",
          "facets": ["sport", "has_route"]
        }
      }
    }
  }
}
```

Launch with:
```bash
datasette polar.db -m metadata.json
```

### 3.5 Relevant Plugins for Personal Data

**Installation:**
```bash
datasette install datasette-dashboards
datasette install datasette-plot
datasette install datasette-vega
```

#### datasette-dashboards

Create interactive dashboards via YAML configuration:

```yaml
# metadata.yml
databases:
  polar:
    dashboards:
      training_overview:
        title: Training Overview
        layout:
          - [training_volume, hr_zones]
          - [activities_by_sport]
        charts:
          training_volume:
            db: polar
            query: |
              SELECT DATE(start_time) as date,
                     SUM(duration)/3600.0 as hours
              FROM exercises
              WHERE start_time >= date('now', '-30 days')
              GROUP BY DATE(start_time)
            library: vega-lite
            mark: bar
            encoding:
              x: {field: date, type: temporal}
              y: {field: hours, type: quantitative}
```

**Features:**
- Grid layouts
- Multiple chart types (vega, vega-lite, markdown, metrics)
- SQL-based data sources
- No JavaScript required

#### datasette-vega

Embed Vega-Lite visualizations directly in custom SQL queries:

```sql
SELECT * FROM exercises ORDER BY start_time DESC
```

Then add visualization via UI or metadata.

#### datasette-plot

Simpler charting plugin:

```bash
# URL-based charting
/database/table.plot?x=date&y=value&type=line
```

### 3.6 Publishing (for future reference)

```bash
# Publish to Vercel
datasette publish vercel mydata.db

# Publish to Google Cloud Run
datasette publish cloudrun mydata.db --service=my-data

# Install plugins in published instance
datasette publish vercel mydata.db --install=datasette-dashboards
```

---

## 4. The Polar AccessLink API

### 4.1 Overview

**Base URLs:**
- Authorization: `https://flow.polar.com/oauth2/authorization`
- Token exchange: `https://polarremote.com/v2/oauth2/token`
- API base: `https://www.polaraccesslink.com/v3`

**Authentication:** OAuth 2.0 Authorization Code flow

**Data retention:** Only exercises/data uploaded to Flow in the **last 30 days** are available via API. This is a critical limitation.

### 4.2 Setup Requirements

1. Go to https://admin.polaraccesslink.com
2. Create OAuth2 client
3. Note `client_id` and `client_secret`
4. Configure redirect URI (e.g., `http://localhost:5000/oauth/callback`)

### 4.3 OAuth2 Flow

Based on the official Python example (`accesslink/oauth2.py`):

```python
class OAuth2Client:
    def __init__(self, client_id, client_secret, redirect_url):
        self.client_id = client_id
        self.client_secret = client_secret
        self.redirect_url = redirect_url
        self.authorization_url = "https://flow.polar.com/oauth2/authorization"
        self.access_token_url = "https://polarremote.com/v2/oauth2/token"

    def get_authorization_url(self):
        """Build URL to redirect user to"""
        params = {
            "client_id": self.client_id,
            "response_type": "code",
        }
        if self.redirect_url:
            params["redirect_uri"] = self.redirect_url

        return f"{self.authorization_url}?{urlencode(params)}"

    def get_access_token(self, authorization_code):
        """Exchange auth code for access token"""
        data = {
            "grant_type": "authorization_code",
            "code": authorization_code,
        }
        if self.redirect_url:
            data["redirect_uri"] = self.redirect_url

        response = requests.post(
            self.access_token_url,
            data=data,
            auth=HTTPBasicAuth(self.client_id, self.client_secret),
            headers={
                "Content-Type": "application/x-www-form-urlencoded",
                "Accept": "application/json;charset=UTF-8"
            }
        )
        return response.json()  # {"access_token": "...", "token_type": "Bearer", ...}
```

**Response contains:**
```json
{
  "access_token": "eyJhbGc...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "x_user_id": 12345678
}
```

**Subsequent requests:**
```python
headers = {
    "Authorization": f"Bearer {access_token}",
    "Accept": "application/json"
}
response = requests.get(f"{API_BASE}/users/{user_id}/...", headers=headers)
```

### 4.4 Available Endpoints and Data Types

#### User Registration

```http
POST /v3/users
Authorization: Basic {client_credentials}
Content-Type: application/json

{"member-id": "user-uuid"}
```

**Response:**
```json
{
  "polar-user-id": 12345678,
  "member-id": "user-uuid",
  "registration-date": "2023-01-15T10:30:00"
}
```

**Note:** Must register user with your client before accessing their data.

#### Pull Notifications (Check for New Data)

```http
GET /v3/notifications
Authorization: Basic {client_credentials}
```

**Response:**
```json
{
  "available-user-data": [
    {
      "user-id": 12345678,
      "data-type": "EXERCISE"
    },
    {
      "user-id": 12345678,
      "data-type": "ACTIVITY_SUMMARY"
    },
    {
      "user-id": 12345678,
      "data-type": "PHYSICAL_INFORMATION"
    }
  ]
}
```

**Data types:** `EXERCISE`, `ACTIVITY_SUMMARY`, `PHYSICAL_INFORMATION`, `SLEEP`, `RECHARGE`

#### Training Data (Exercises) - Transaction-based

The Polar API uses a **transaction pattern** for retrieving training data:

**Step 1: Create transaction**
```http
POST /v3/users/{user_id}/exercise-transactions
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "transaction-id": 179879,
  "resource-uri": "https://www.polaraccesslink.com/v3/users/12345678/exercise-transactions/179879"
}
```

Returns `null` if no new data.

**Step 2: List exercises in transaction**
```http
GET /v3/users/{user_id}/exercise-transactions/{transaction_id}
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "exercises": [
    "https://www.polaraccesslink.com/.../exercises/ABC123",
    "https://www.polaraccesslink.com/.../exercises/DEF456"
  ]
}
```

Maximum 50 exercises per transaction.

**Step 3: Get exercise details**
```http
GET {exercise_url}
Authorization: Bearer {access_token}
```

**Response (exercise summary):**
```json
{
  "id": 12345,
  "upload-time": "2023-03-09T08:30:00Z",
  "polar-user": "https://...",
  "transaction-id": 179879,
  "device": "Polar Vantage V",
  "device-id": "1234ABCD",
  "start-time": "2023-03-09T06:00:00Z",
  "duration": "PT1H15M30S",
  "calories": 850,
  "distance": 10500.0,
  "heart-rate": {
    "average": 142,
    "maximum": 178
  },
  "training-load": 95.5,
  "sport": "RUNNING",
  "has-route": true,
  "club-id": null,
  "detailed-sport-info": "ROAD_RUNNING",
  "fat-percentage": 45,
  "carbohydrate-percentage": 55,
  "protein-percentage": 0,
  "running-index": 58,
  "training-load-pro": [
    {
      "date": "2023-03-09",
      "cardio-load": 42.5,
      "muscle-load": 28.3,
      "perceived-load": 24.2,
      "cardio-load-interpretation": "MODERATE",
      "muscle-load-interpretation": "MODERATE"
    }
  ]
}
```

**Step 4: Get samples (time-series data)**
```http
GET {exercise_url}/samples
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "samples": [
    "https://www.polaraccesslink.com/.../samples/0",  // Heart rate
    "https://www.polaraccesslink.com/.../samples/1",  // Speed
    "https://www.polaraccesslink.com/.../samples/2"   // Cadence
  ]
}
```

**Sample types:**
- `0` = Heart rate
- `1` = Speed
- `2` = Cadence
- `3` = Altitude
- `4` = Power
- `5` = Stride length
- (more types for different sports)

**Fetch individual sample:**
```http
GET {sample_url}
```

**Response:**
```json
{
  "recording-rate": 1,  // Hz
  "sample-type": "0",   // Heart rate
  "data": "60,62,65,68,72,75,78,82,..."  // Comma-separated values
}
```

**Step 5: Get heart rate zones**
```http
GET {exercise_url}/heart-rate-zones
Authorization: Bearer {access_token}
```

**Response:**
```json
[
  {
    "index": 0,
    "lower-limit": 100,
    "upper-limit": 120,
    "in-zone": "PT5M30S"
  },
  {
    "index": 1,
    "lower-limit": 120,
    "upper-limit": 140,
    "in-zone": "PT25M15S"
  },
  ...
]
```

**Step 6: Get TCX/GPX export**
```http
GET {exercise_url}/tcx
Accept: application/vnd.garmin.tcx+xml

GET {exercise_url}/gpx
Accept: application/gpx+xml
```

Returns XML file with full workout data.

**Step 7: Commit transaction**
```http
DELETE /v3/users/{user_id}/exercise-transactions/{transaction_id}
Authorization: Bearer {access_token}
```

**Critical:** Must commit transaction to mark data as retrieved. Uncommitted transactions will return same data on next fetch.

#### Code example from official client:

```python
# From accesslink/endpoints/training_data_transaction.py
class TrainingDataTransaction:
    def list_exercises(self):
        return self._get(url=self.transaction_url, access_token=self.access_token)

    def get_exercise_summary(self, url):
        return self._get(url=url, access_token=self.access_token)

    def get_samples(self, url):
        return self._get(url=url, access_token=self.access_token)

    def get_heart_rate_zones(self, url):
        return self._get(url=url+"/heart-rate-zones", access_token=self.access_token)

    def get_gpx(self, url):
        return self._get(
            url=url+"/gpx",
            access_token=self.access_token,
            headers={"Accept": "application/gpx+xml"}
        )

    def get_tcx(self, url):
        return self._get(
            url=url+"/tcx",
            access_token=self.access_token,
            headers={"Accept": "application/vnd.garmin.tcx+xml"}
        )

    def commit(self):
        return self._delete(url=self.transaction_url, access_token=self.access_token)
```

#### Daily Activity

Similar transaction pattern:

```http
POST /v3/users/{user_id}/activity-transactions
GET /v3/users/{user_id}/activity-transactions/{transaction_id}
GET {activity_url}
DELETE /v3/users/{user_id}/activity-transactions/{transaction_id}
```

**Activity summary structure:**
```json
{
  "id": "2023-03-09",
  "date": "2023-03-09",
  "created": "2023-03-10T02:00:00Z",
  "calories": 2100,
  "active-calories": 450,
  "duration": "PT14H30M",
  "active-steps": 8500,
  "activity-distance": 6.2,
  "bmr": 1650
}
```

#### Sleep Data

```http
GET /v3/users/sleep/
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "nights": [
    {
      "polar-user": "...",
      "date": "2023-03-08",
      "sleep-start-time": "2023-03-08T23:15:00",
      "sleep-end-time": "2023-03-09T07:30:00",
      "continuity": 3.5,
      "continuity-class": 3,
      "light-sleep": "PT4H25M",
      "deep-sleep": "PT2H10M",
      "rem-sleep": "PT1H30M",
      "unrecognized-sleep-stage": "PT0M",
      "sleep-score": 75,
      "total-interruption-duration": "PT35M",
      "sleep-charge": 85,
      "sleep-rating": 4,
      "short-interruption": 12,
      "long-interruption": 2,
      "sleep-cycles": 5,
      "group-duration-score": 85,
      "group-solidity-score": 70,
      "group-regeneration-score": 75
    }
  ]
}
```

#### Nightly Recharge (Recovery)

```http
GET /v3/users/nightly-recharge/
Authorization: Bearer {access_token}
```

**Response:**
```json
{
  "recharges": [
    {
      "polar-user": "...",
      "date": "2023-03-09",
      "nightly-recharge-status": 3,
      "ans-charge": 2,
      "ans-charge-status": 2,
      "hrv-avg": 45,
      "breathing-rate": 14.5,
      "heart-rate-avg": 52,
      "sleep-charge": 85,
      "sleep-charge-status": 4
    }
  ]
}
```

#### Physical Information (Body metrics)

Transaction-based like exercises:

```http
POST /v3/users/{user_id}/physical-information-transactions
GET /v3/users/{user_id}/physical-information-transactions/{transaction_id}
DELETE /v3/users/{user_id}/physical-information-transactions/{transaction_id}
```

**Physical info structure:**
```json
{
  "transaction-id": 123,
  "weight": 72.5,
  "height": 180.0,
  "maximum-heart-rate": 188,
  "resting-heart-rate": 55,
  "aerobic-threshold": 160,
  "anaerobic-threshold": 175,
  "vo2-max": 58
}
```

### 4.5 Rate Limits and Pagination

**Rate limits:** Not explicitly documented, but API examples use `time.sleep(1)` between requests.

**Pagination:**
- Exercises: Max 50 per transaction
- Other data: Returned in single response (last 30 days)

**Best practice:** Always include delays between requests.

### 4.6 Existing Python Implementations

**1. Official example:** `polarofficial/accesslink-example-python`
- Wrapper client in `accesslink/` directory
- Example console and web apps
- OAuth flow implemented
- Transaction pattern demonstrated

**2. PyPI package:** `polar-accesslink` (v0.0.5, 2019)
- Packaged version of official example
- **Inactive maintenance** (last update 2019)
- Not recommended for new projects

**Recommendation:** Vendor the official `accesslink/` directory or implement minimal wrapper inspired by it.

---

## 5. Proposed Design for polar-to-sqlite

Based on all findings above, here's the concrete design for `polar-to-sqlite`.

### 5.1 Project Structure

```
polar-to-sqlite/
├── polar_to_sqlite/
│   ├── __init__.py
│   ├── cli.py              # Click commands
│   ├── utils.py            # Database save functions
│   ├── polar.py            # API wrapper (vendored/adapted from official)
│   └── oauth_server.py     # Local server for OAuth callback (optional)
├── tests/
│   ├── fixtures/
│   │   ├── exercise.json
│   │   ├── samples.json
│   │   ├── sleep.json
│   │   └── activity.json
│   ├── test_exercises.py
│   ├── test_sleep.py
│   └── test_activities.py
├── pyproject.toml
├── README.md
└── CLAUDE.md
```

### 5.2 Packaging (pyproject.toml)

```toml
[build-system]
requires = ["setuptools>=61.0"]
build-backend = "setuptools.build_meta"

[project]
name = "polar-to-sqlite"
version = "0.1.0"
description = "Save training data from Polar AccessLink API to SQLite"
readme = "README.md"
requires-python = ">=3.8"
license = {text = "Apache-2.0"}
authors = [
    {name = "Your Name", email = "you@example.com"}
]
dependencies = [
    "sqlite-utils>=3.30",
    "click>=8.0",
    "requests>=2.28",
]

[project.optional-dependencies]
test = [
    "pytest>=7.0",
    "requests-mock>=1.9",
]

[project.scripts]
polar-to-sqlite = "polar_to_sqlite.cli:cli"

[project.urls]
Homepage = "https://github.com/yourusername/polar-to-sqlite"
Issues = "https://github.com/yourusername/polar-to-sqlite/issues"
```

**Rationale:** Modern `pyproject.toml` over `setup.py`, following current Python packaging standards.

### 5.3 CLI Design

```python
# polar_to_sqlite/cli.py
import click
import sqlite_utils
from .polar import PolarClient
from .utils import (
    save_exercise,
    save_activity,
    save_sleep,
    save_physical_info,
    ensure_db_shape,
)

@click.group()
@click.version_option()
def cli():
    """Save data from Polar AccessLink API to SQLite database"""


@cli.command()
@click.option("--client-id", required=True, help="Polar OAuth client ID")
@click.option("--client-secret", required=True, help="Polar OAuth client secret")
@click.option("-a", "--auth", default="auth.json", help="Path to auth file")
def auth(client_id, client_secret, auth):
    """Authenticate with Polar and save credentials"""
    # 1. Start local server on random port for OAuth callback
    # 2. Open browser to authorization URL
    # 3. Receive callback with auth code
    # 4. Exchange for access token
    # 5. Register user if not registered
    # 6. Save to auth.json: {client_id, client_secret, access_token, user_id}
    pass


@cli.command()
@click.argument("db_path", type=click.Path(...), required=True)
@click.option("-a", "--auth", default="auth.json")
@click.option("--all", is_flag=True, help="Fetch all available data (ignore existing)")
def exercises(db_path, auth, all):
    """Fetch exercise/workout data"""
    config = load_config(auth)
    client = PolarClient(
        client_id=config["client_id"],
        client_secret=config["client_secret"],
    )
    db = sqlite_utils.Database(db_path)

    # Create transaction
    transaction = client.create_exercise_transaction(
        user_id=config["user_id"],
        access_token=config["access_token"]
    )

    if not transaction:
        click.echo("No new exercises available")
        return

    exercise_urls = transaction.list_exercises()

    with click.progressbar(exercise_urls, label="Fetching exercises") as bar:
        for url in bar:
            # Fetch summary
            summary = transaction.get_exercise_summary(url)

            # Fetch samples if route exists
            samples = []
            if summary.get("has-route"):
                sample_urls = transaction.get_available_samples(url)
                for sample_url in sample_urls:
                    samples.append(transaction.get_samples(sample_url))

            # Fetch HR zones
            zones = transaction.get_heart_rate_zones(url)

            # Save to database
            save_exercise(db, summary, samples, zones)

    transaction.commit()
    ensure_db_shape(db)


@cli.command()
@click.argument("db_path", type=click.Path(...), required=True)
@click.option("-a", "--auth", default="auth.json")
def activities(db_path, auth):
    """Fetch daily activity summaries"""
    # Similar pattern to exercises
    pass


@cli.command()
@click.argument("db_path", type=click.Path(...), required=True)
@click.option("-a", "--auth", default="auth.json")
def sleep(db_path, auth):
    """Fetch sleep data"""
    config = load_config(auth)
    client = PolarClient(...)
    db = sqlite_utils.Database(db_path)

    sleep_data = client.get_sleep(access_token=config["access_token"])

    for night in sleep_data.get("nights", []):
        save_sleep(db, night)

    ensure_db_shape(db)


@cli.command()
@click.argument("db_path", type=click.Path(...), required=True)
@click.option("-a", "--auth", default="auth.json")
def sync(db_path, auth):
    """Sync all available data types"""
    ctx = click.get_current_context()

    # Invoke other commands
    ctx.invoke(exercises, db_path=db_path, auth=auth)
    ctx.invoke(activities, db_path=db_path, auth=auth)
    ctx.invoke(sleep, db_path=db_path, auth=auth)

    click.echo("✓ Sync complete")
```

**Commands:**
- `polar-to-sqlite auth` - OAuth flow, save credentials
- `polar-to-sqlite exercises polar.db` - Fetch workouts
- `polar-to-sqlite activities polar.db` - Fetch daily summaries
- `polar-to-sqlite sleep polar.db` - Fetch sleep data
- `polar-to-sqlite sync polar.db` - Fetch everything

### 5.4 Proposed Table Schemas

#### Exercises (workouts)

```sql
CREATE TABLE exercises (
    id INTEGER PRIMARY KEY,
    polar_user TEXT,
    upload_time TEXT,
    start_time TEXT NOT NULL,
    duration TEXT,  -- ISO8601 duration
    duration_seconds INTEGER,  -- Computed for querying
    calories INTEGER,
    distance REAL,
    heart_rate_avg INTEGER,
    heart_rate_max INTEGER,
    training_load REAL,
    sport TEXT,
    detailed_sport TEXT,
    device TEXT,
    device_id TEXT,
    has_route BOOLEAN,
    running_index INTEGER,
    fat_percentage REAL,
    carbohydrate_percentage REAL,
    protein_percentage REAL,
    transaction_id INTEGER,
    raw_data TEXT  -- JSON blob of full API response
);

CREATE INDEX idx_exercises_start_time ON exercises(start_time);
CREATE INDEX idx_exercises_sport ON exercises(sport);
```

#### Training Load Pro (nested in exercise)

```sql
CREATE TABLE training_load_pro (
    id INTEGER PRIMARY KEY,
    exercise_id INTEGER NOT NULL,
    date TEXT,
    cardio_load REAL,
    muscle_load REAL,
    perceived_load REAL,
    cardio_load_interpretation TEXT,
    muscle_load_interpretation TEXT,
    FOREIGN KEY (exercise_id) REFERENCES exercises(id)
);
```

#### Heart Rate Zones

```sql
CREATE TABLE heart_rate_zones (
    id INTEGER PRIMARY KEY,
    exercise_id INTEGER NOT NULL,
    zone_index INTEGER,
    lower_limit INTEGER,
    upper_limit INTEGER,
    in_zone_seconds INTEGER,
    FOREIGN KEY (exercise_id) REFERENCES exercises(id)
);
```

#### Exercise Samples (time-series)

```sql
CREATE TABLE exercise_samples (
    id INTEGER PRIMARY KEY,
    exercise_id INTEGER NOT NULL,
    sample_type INTEGER,  -- 0=HR, 1=speed, 2=cadence, etc.
    sample_type_name TEXT,
    recording_rate INTEGER,
    data TEXT,  -- Comma-separated values (keep as-is or parse to JSON array?)
    FOREIGN KEY (exercise_id) REFERENCES exercises(id)
);

CREATE INDEX idx_samples_exercise ON exercise_samples(exercise_id);
```

**Design decision:** Store sample data as TEXT (comma-separated) rather than one row per sample point. Rationale: Can have 3600+ points per hour; storing individually would create millions of rows. Better to keep CSV and parse in application/Datasette query.

#### Daily Activities

```sql
CREATE TABLE daily_activities (
    date TEXT PRIMARY KEY,
    created TEXT,
    calories INTEGER,
    active_calories INTEGER,
    duration_seconds INTEGER,
    active_steps INTEGER,
    activity_distance REAL,
    bmr INTEGER,
    raw_data TEXT
);
```

#### Sleep

```sql
CREATE TABLE sleep (
    id INTEGER PRIMARY KEY,
    date TEXT UNIQUE,
    sleep_start_time TEXT,
    sleep_end_time TEXT,
    duration_seconds INTEGER,
    continuity REAL,
    continuity_class INTEGER,
    light_sleep_seconds INTEGER,
    deep_sleep_seconds INTEGER,
    rem_sleep_seconds INTEGER,
    unrecognized_sleep_seconds INTEGER,
    sleep_score INTEGER,
    sleep_charge INTEGER,
    sleep_rating INTEGER,
    total_interruption_seconds INTEGER,
    short_interruption INTEGER,
    long_interruption INTEGER,
    sleep_cycles INTEGER,
    group_duration_score INTEGER,
    group_solidity_score INTEGER,
    group_regeneration_score INTEGER,
    raw_data TEXT
);

CREATE INDEX idx_sleep_date ON sleep(date);
```

#### Nightly Recharge

```sql
CREATE TABLE nightly_recharge (
    id INTEGER PRIMARY KEY,
    date TEXT UNIQUE,
    nightly_recharge_status INTEGER,
    ans_charge INTEGER,
    ans_charge_status INTEGER,
    hrv_avg INTEGER,
    breathing_rate REAL,
    heart_rate_avg INTEGER,
    sleep_charge INTEGER,
    sleep_charge_status INTEGER,
    raw_data TEXT
);
```

#### Physical Information

```sql
CREATE TABLE physical_info (
    id INTEGER PRIMARY KEY,
    transaction_id INTEGER,
    created TEXT,
    weight REAL,
    height REAL,
    maximum_heart_rate INTEGER,
    resting_heart_rate INTEGER,
    aerobic_threshold INTEGER,
    anaerobic_threshold INTEGER,
    vo2_max INTEGER
);
```

### 5.5 Database Writing Functions

```python
# polar_to_sqlite/utils.py
import json
from datetime import datetime

def parse_iso8601_duration(duration_str):
    """Convert PT1H15M30S to seconds"""
    # Use isodate library or simple regex
    import re
    pattern = r'PT(?:(\d+)H)?(?:(\d+)M)?(?:(\d+)S)?'
    match = re.match(pattern, duration_str)
    if not match:
        return None
    h, m, s = match.groups()
    return (int(h or 0) * 3600 + int(m or 0) * 60 + int(s or 0))


def save_exercise(db, summary, samples, zones):
    """Save exercise and related data"""
    # Transform summary
    exercise = {
        "id": summary["id"],
        "polar_user": summary.get("polar-user"),
        "upload_time": summary.get("upload-time"),
        "start_time": summary["start-time"],
        "duration": summary.get("duration"),
        "duration_seconds": parse_iso8601_duration(summary.get("duration")),
        "calories": summary.get("calories"),
        "distance": summary.get("distance"),
        "heart_rate_avg": summary.get("heart-rate", {}).get("average"),
        "heart_rate_max": summary.get("heart-rate", {}).get("maximum"),
        "training_load": summary.get("training-load"),
        "sport": summary.get("sport"),
        "detailed_sport": summary.get("detailed-sport-info"),
        "device": summary.get("device"),
        "device_id": summary.get("device-id"),
        "has_route": summary.get("has-route", False),
        "running_index": summary.get("running-index"),
        "fat_percentage": summary.get("fat-percentage"),
        "carbohydrate_percentage": summary.get("carbohydrate-percentage"),
        "protein_percentage": summary.get("protein-percentage"),
        "transaction_id": summary.get("transaction-id"),
        "raw_data": json.dumps(summary),  # Keep full response
    }

    # Insert exercise
    exercise_table = db["exercises"].insert(
        exercise,
        pk="id",
        replace=True,
        alter=True,
        columns={
            "start_time": str,
            "sport": str,
            "has_route": bool,
        }
    )
    exercise_id = exercise_table.last_pk

    # Save training load pro (nested array)
    training_load_pro = summary.get("training-load-pro", [])
    for tlp in training_load_pro:
        db["training_load_pro"].insert(
            {
                "exercise_id": exercise_id,
                "date": tlp.get("date"),
                "cardio_load": tlp.get("cardio-load"),
                "muscle_load": tlp.get("muscle-load"),
                "perceived_load": tlp.get("perceived-load"),
                "cardio_load_interpretation": tlp.get("cardio-load-interpretation"),
                "muscle_load_interpretation": tlp.get("muscle-load-interpretation"),
            },
            alter=True,
            foreign_keys=[("exercise_id", "exercises", "id")],
        )

    # Save heart rate zones
    for zone in zones:
        in_zone_seconds = parse_iso8601_duration(zone.get("in-zone"))
        db["heart_rate_zones"].insert(
            {
                "exercise_id": exercise_id,
                "zone_index": zone.get("index"),
                "lower_limit": zone.get("lower-limit"),
                "upper_limit": zone.get("upper-limit"),
                "in_zone_seconds": in_zone_seconds,
            },
            alter=True,
            foreign_keys=[("exercise_id", "exercises", "id")],
        )

    # Save samples
    for sample in samples:
        sample_type = int(sample["sample-type"])
        sample_type_names = {
            0: "heart_rate",
            1: "speed",
            2: "cadence",
            3: "altitude",
            4: "power",
            5: "stride_length",
        }

        db["exercise_samples"].insert(
            {
                "exercise_id": exercise_id,
                "sample_type": sample_type,
                "sample_type_name": sample_type_names.get(sample_type, f"type_{sample_type}"),
                "recording_rate": sample.get("recording-rate"),
                "data": sample.get("data"),  # Keep as CSV string
            },
            alter=True,
            foreign_keys=[("exercise_id", "exercises", "id")],
        )


def save_sleep(db, night):
    """Save sleep data"""
    def parse_duration(d):
        return parse_iso8601_duration(d) if d else None

    sleep_record = {
        "date": night["date"],
        "sleep_start_time": night.get("sleep-start-time"),
        "sleep_end_time": night.get("sleep-end-time"),
        "continuity": night.get("continuity"),
        "continuity_class": night.get("continuity-class"),
        "light_sleep_seconds": parse_duration(night.get("light-sleep")),
        "deep_sleep_seconds": parse_duration(night.get("deep-sleep")),
        "rem_sleep_seconds": parse_duration(night.get("rem-sleep")),
        "unrecognized_sleep_seconds": parse_duration(night.get("unrecognized-sleep-stage")),
        "sleep_score": night.get("sleep-score"),
        "sleep_charge": night.get("sleep-charge"),
        "sleep_rating": night.get("sleep-rating"),
        "total_interruption_seconds": parse_duration(night.get("total-interruption-duration")),
        "short_interruption": night.get("short-interruption"),
        "long_interruption": night.get("long-interruption"),
        "sleep_cycles": night.get("sleep-cycles"),
        "group_duration_score": night.get("group-duration-score"),
        "group_solidity_score": night.get("group-solidity-score"),
        "group_regeneration_score": night.get("group-regeneration-score"),
        "raw_data": json.dumps(night),
    }

    # Compute duration if not provided
    if night.get("sleep-start-time") and night.get("sleep-end-time"):
        start = datetime.fromisoformat(night["sleep-start-time"].replace("Z", "+00:00"))
        end = datetime.fromisoformat(night["sleep-end-time"].replace("Z", "+00:00"))
        sleep_record["duration_seconds"] = int((end - start).total_seconds())

    db["sleep"].upsert(
        sleep_record,
        pk="date",
        alter=True,
    )


def save_activity(db, activity):
    """Save daily activity"""
    db["daily_activities"].upsert(
        {
            "date": activity["date"],
            "created": activity.get("created"),
            "calories": activity.get("calories"),
            "active_calories": activity.get("active-calories"),
            "duration_seconds": parse_iso8601_duration(activity.get("duration")),
            "active_steps": activity.get("active-steps"),
            "activity_distance": activity.get("activity-distance"),
            "bmr": activity.get("bmr"),
            "raw_data": json.dumps(activity),
        },
        pk="date",
        alter=True,
    )


def ensure_db_shape(db):
    """Finalize schema: indexes, FTS, views"""
    # Add indexes
    if "exercises" in db.table_names():
        db["exercises"].create_index(["start_time"], if_not_exists=True)
        db["exercises"].create_index(["sport"], if_not_exists=True)
        db["exercises"].create_index(["has_route"], if_not_exists=True)

    # Enable FTS on exercise raw data (for searching notes, etc.)
    # Note: Polar doesn't have notes field, but might in future

    # Create useful views
    if {"exercises", "heart_rate_zones"}.issubset(set(db.table_names())):
        db.create_view(
            "exercise_summary",
            """
            SELECT
                e.id,
                e.start_time,
                e.sport,
                e.detailed_sport,
                e.duration_seconds / 3600.0 as duration_hours,
                e.distance / 1000.0 as distance_km,
                e.calories,
                e.heart_rate_avg,
                e.training_load,
                COUNT(DISTINCT hrz.zone_index) as num_hr_zones
            FROM exercises e
            LEFT JOIN heart_rate_zones hrz ON e.id = hrz.exercise_id
            GROUP BY e.id
            ORDER BY e.start_time DESC
            """,
            replace=True
        )

    if "sleep" in db.table_names():
        db.create_view(
            "sleep_summary",
            """
            SELECT
                date,
                sleep_start_time,
                duration_seconds / 3600.0 as duration_hours,
                sleep_score,
                sleep_charge,
                deep_sleep_seconds / 60.0 as deep_sleep_minutes,
                rem_sleep_seconds / 60.0 as rem_sleep_minutes,
                sleep_cycles
            FROM sleep
            ORDER BY date DESC
            """,
            replace=True
        )
```

### 5.6 Auth Flow Implementation

**Two approaches:**

#### Approach A: Manual (like github-to-sqlite)

```python
@cli.command()
def auth(client_id, client_secret, auth):
    """Authenticate with Polar"""
    client = PolarClient(client_id, client_secret, redirect_url=None)

    auth_url = client.get_authorization_url()

    click.echo("Visit this URL to authorize:")
    click.echo(auth_url)
    click.echo()
    click.echo("After authorizing, you'll be redirected to a URL like:")
    click.echo("  http://localhost/?code=XXXXXX")
    click.echo()

    auth_code = click.prompt("Paste the 'code' parameter here")

    # Exchange for token
    token_response = client.get_access_token(auth_code)
    access_token = token_response["access_token"]
    user_id = token_response["x_user_id"]

    # Register user (if not already)
    # Polar requires explicit user registration
    try:
        client.register_user(user_id)
    except Exception as e:
        # May already be registered
        pass

    # Save credentials
    config = {
        "client_id": client_id,
        "client_secret": client_secret,
        "access_token": access_token,
        "user_id": user_id,
    }

    with open(auth, "w") as f:
        json.dump(config, f, indent=2)

    click.echo(f"✓ Credentials saved to {auth}")
```

**User experience:**
1. Run `polar-to-sqlite auth --client-id=... --client-secret=...`
2. Copy/paste URL into browser
3. Authorize app
4. Copy code from redirect URL
5. Paste back into terminal

**Pro:** No dependencies, no server needed
**Con:** Manual copy/paste

#### Approach B: Local server (like official example)

```python
# polar_to_sqlite/oauth_server.py
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import urlparse, parse_qs
import threading
import webbrowser

class OAuthCallbackHandler(BaseHTTPRequestHandler):
    auth_code = None

    def do_GET(self):
        query = parse_qs(urlparse(self.path).query)

        if "code" in query:
            OAuthCallbackHandler.auth_code = query["code"][0]
            self.send_response(200)
            self.send_header("Content-type", "text/html")
            self.end_headers()
            self.wfile.write(b"<h1>Authorization successful!</h1><p>You can close this window.</p>")
        else:
            self.send_response(400)
            self.end_headers()

    def log_message(self, format, *args):
        pass  # Suppress logs

def run_auth_server(port=5000):
    """Start server, return auth code when received"""
    server = HTTPServer(("localhost", port), OAuthCallbackHandler)
    server.timeout = 300  # 5 minutes

    # Wait for single request
    server.handle_request()

    return OAuthCallbackHandler.auth_code


@cli.command()
def auth(client_id, client_secret, auth):
    """Authenticate with Polar (opens browser)"""
    redirect_url = "http://localhost:5000/oauth/callback"
    client = PolarClient(client_id, client_secret, redirect_url)

    auth_url = client.get_authorization_url()

    click.echo("Opening browser for authorization...")
    click.echo("If browser doesn't open, visit:")
    click.echo(auth_url)

    webbrowser.open(auth_url)

    # Start local server
    auth_code = run_auth_server(port=5000)

    if not auth_code:
        raise click.ClickException("Authorization timed out")

    # Exchange for token
    # ... (same as Approach A)
```

**Pro:** Better UX, no copy/paste
**Con:** Requires configuring redirect URI in Polar admin, port may be in use

**Recommendation:** Start with Approach A (simpler), add Approach B later if desired.

### 5.7 Incremental Sync Strategy

**Challenge:** Polar API only returns last 30 days of data. Can't use "fetch all then stop when seen" pattern like github-to-sqlite.

**Strategy:**

1. **Transaction-based data (exercises, activities, physical_info):**
   - API automatically only returns new data per transaction
   - Commit transaction after successful save
   - Next run gets only new data
   - **No additional logic needed**

2. **Non-transactional data (sleep, recharge):**
   - API returns all records from last 30 days
   - Check if record exists before inserting:

   ```python
   def save_sleep(db, night):
       # Check if already exists
       existing = list(db["sleep"].rows_where("date = ?", [night["date"]]))
       if existing and not force_update:
           return  # Skip

       # Otherwise upsert
       db["sleep"].upsert({...}, pk="date")
   ```

3. **Handling 30-day window:**
   - Document limitation in README
   - Recommend running sync at least once every 30 days
   - Consider adding `--backfill` flag that exports TCX/GPX for archival

**Note:** No true "incremental" in the GitHub sense, but transaction pattern provides similar benefit.

### 5.8 Testing Approach

```python
# tests/test_exercises.py
import pytest
import sqlite_utils
import json
from pathlib import Path
from polar_to_sqlite.utils import save_exercise, ensure_db_shape

@pytest.fixture
def exercise_fixture():
    """Load exercise JSON fixture"""
    fixture_path = Path(__file__).parent / "fixtures" / "exercise.json"
    return json.load(open(fixture_path))

@pytest.fixture
def samples_fixture():
    return [
        {
            "sample-type": "0",
            "recording-rate": 1,
            "data": "60,62,65,68,72,75,78"
        }
    ]

@pytest.fixture
def zones_fixture():
    return [
        {
            "index": 0,
            "lower-limit": 100,
            "upper-limit": 120,
            "in-zone": "PT5M30S"
        },
        {
            "index": 1,
            "lower-limit": 120,
            "upper-limit": 140,
            "in-zone": "PT25M15S"
        }
    ]

@pytest.fixture
def db(exercise_fixture, samples_fixture, zones_fixture):
    """In-memory DB with test data"""
    db = sqlite_utils.Database(memory=True)
    save_exercise(db, exercise_fixture, samples_fixture, zones_fixture)
    ensure_db_shape(db)
    return db

def test_tables_created(db):
    assert "exercises" in db.table_names()
    assert "heart_rate_zones" in db.table_names()
    assert "exercise_samples" in db.table_names()

def test_exercise_data(db):
    exercise = list(db["exercises"].rows)[0]
    assert exercise["sport"] == "RUNNING"
    assert exercise["duration_seconds"] > 0

def test_zones(db):
    zones = list(db["heart_rate_zones"].rows)
    assert len(zones) == 2
    assert zones[0]["lower_limit"] == 100

def test_samples(db):
    samples = list(db["exercise_samples"].rows)
    assert len(samples) == 1
    assert samples[0]["sample_type_name"] == "heart_rate"

def test_views_created(db):
    assert "exercise_summary" in db.view_names()
```

**Testing strategy:**
- JSON fixtures from real API responses (anonymized)
- In-memory databases for fast tests
- Test both data transformation and schema creation
- Mock API calls using `requests-mock`

### 5.9 Example Datasette Configuration

```yaml
# metadata.yml
title: Polar Training Data
description: Personal training data from Polar Flow
databases:
  polar:
    tables:
      exercises:
        sort_desc: start_time
        facets:
          - sport
          - has_route
          - detailed_sport
        label_column: start_time
      sleep:
        sort_desc: date
        label_column: date
      exercise_summary:
        sort_desc: start_time
    queries:
      recent_runs:
        sql: |
          SELECT
            start_time,
            distance_km,
            duration_hours,
            heart_rate_avg,
            training_load
          FROM exercise_summary
          WHERE sport = 'RUNNING'
          ORDER BY start_time DESC
          LIMIT 20
```

**Launch:**
```bash
datasette polar.db -m metadata.yml --install datasette-dashboards
```

---

## Key Design Decisions

### 1. Vendor vs. Depend on polar-accesslink PyPI Package

**Decision:** Vendor a minimal API wrapper based on official example.

**Rationale:**
- PyPI package is unmaintained (last update 2019)
- Official example is up-to-date
- Minimal code needed (~200 lines)
- Full control over dependencies

### 2. Store Samples as CSV String vs. Parse to Rows

**Decision:** Keep samples as TEXT (CSV string) initially.

**Rationale:**
- A 1-hour run can have 3600+ HR samples
- Storing each as a row = millions of rows quickly
- SQLite has 35TB row limit but performance degrades
- Can always parse in application layer or Datasette query
- If analysis needed, can add `parse_samples` command later

### 3. Auth Flow: Manual vs. Local Server

**Decision:** Start with manual copy/paste, add local server later.

**Rationale:**
- Simpler implementation
- No port conflicts
- Easier to test
- Can enhance later without breaking changes

### 4. Raw Data Storage

**Decision:** Store full API response as JSON in `raw_data` TEXT column.

**Rationale:**
- API may add fields in future
- Debugging aid
- Allows re-processing without re-fetching
- Disk is cheap
- Can extract new fields later without API calls

### 5. Duration Format

**Decision:** Store both ISO8601 string and integer seconds.

**Rationale:**
- ISO8601 preserves original API format
- Integer seconds enables SQL aggregation/math
- Computed on save, not on query

### 6. Incremental Sync

**Decision:** Rely on Polar's transaction pattern, no custom logic.

**Rationale:**
- Transaction pattern already handles "new data only"
- 30-day window is API limitation, not solvable
- Document limitation rather than work around it

---

## Next Steps

1. **Implement auth command** - OAuth flow + credential storage
2. **Implement exercises command** - Transaction-based fetch
3. **Implement save_exercise()** - Database writing with FK relationships
4. **Add tests** - Fixtures + in-memory DB
5. **Implement sleep command** - Non-transactional endpoint
6. **Implement activities command** - Daily summaries
7. **Add ensure_db_shape()** - Indexes, views, FTS
8. **Documentation** - README with setup instructions
9. **Datasette config** - metadata.yml with queries
10. **Optional enhancements:**
    - Local server OAuth flow
    - TCX/GPX export for archival
    - Sample parsing command
    - Dashboards configuration

---

## Surprising Findings

1. **30-day data retention:** Major limitation. Can't build historical archive without running sync regularly.

2. **Transaction pattern:** Unusual API design. Most APIs use pagination/timestamps, but Polar uses explicit transactions that must be committed. Ensures data isn't double-counted.

3. **No direct "all exercises" endpoint:** Can't list all exercises, only new ones via transaction. Makes initial backfill impossible.

4. **Sample data format:** CSV strings rather than JSON arrays. Compact but awkward.

5. **OAuth user registration:** Must explicitly register user with client after auth. Extra step not in standard OAuth flow.

6. **Duration format inconsistency:** ISO8601 durations (PT1H30M) require parsing, unlike standard Unix timestamps or ISO datetime strings.

7. **No webhook support:** Must poll API for new data. Notification endpoint exists but requires polling.

8. **sqlite-utils .m2m() magic:** Incredibly powerful for relationships. One line creates junction table with proper FKs.

9. **FTS triggers:** `create_triggers=True` keeps FTS in sync automatically. No manual maintenance needed.

10. **Datasette view support:** Views appear as tables in UI. Great for pre-computing common queries.

---

## References

- Dogsheep tools: https://github.com/dogsheep
- sqlite-utils: https://sqlite-utils.datasette.io/
- Datasette: https://datasette.io/
- Polar AccessLink API: https://www.polar.com/accesslink-api/
- Official Python example: https://github.com/polarofficial/accesslink-example-python
- Click documentation: https://click.palletsprojects.com/
