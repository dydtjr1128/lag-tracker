# Storage and Privacy

This document defines the recording file format, schema, and size budget, and the boundary of information that is never collected.

## Files

| File | Format | Created |
| --- | --- | --- |
| `lag-<start time>.lagtrace` | SQLite database | When recording starts |
| `lag-<start time>.html` | Single-file HTML report | Automatically on exit, or with the `report` command |

- The start time is local time in `YYYYMMDD-HHMMSS` format.
- The default location is `records\` next to the executable. If that folder is not writable, `%LOCALAPPDATA%\lag-tracker\records\` is used. The actual location is always shown on the live screen and the exit screen.
- The `.lagtrace` extension separates recordings from ordinary database files. The content is standard SQLite and opens in other tools.

## Write strategy

- The database opens in WAL mode, and steady samples are written in one transaction every five seconds.
- Incident captures are written as soon as the incident starts.
- On a normal exit the tool checkpoints and merges the WAL into a single file. If an abnormal exit leaves a WAL file behind, the next `report` or `info` run merges it.
- The guidance asks users not to open the file in other programs while recording. Reports can still be built during a recording through a read-only connection.

## Schema

The version is tracked by `schema_version` in the `meta` table. A newer tool must be able to read files from older versions.

```sql
-- Recording information: key-value
CREATE TABLE meta (key TEXT PRIMARY KEY, value TEXT NOT NULL);
-- schema_version, tool_version, started_at_ms, ended_at_ms, mode(normal|admin),
-- os_build, cpu_model, logical_cpus, memory_total_kib, gpu_adapters(JSON),
-- disabled_sources(JSON), clock_anchors(JSON), capture_layout(JSON)

-- Process identity: one row per (PID, creation time)
CREATE TABLE process (
  id INTEGER PRIMARY KEY,
  pid INTEGER NOT NULL,
  created_at_ms INTEGER NOT NULL,
  parent_id INTEGER REFERENCES process(id),
  image TEXT NOT NULL,          -- executable name only
  role TEXT,                    -- Chrome role, NULL otherwise
  first_seen_ms INTEGER NOT NULL,
  exited_at_ms INTEGER,
  UNIQUE (pid, created_at_ms)
);

-- Steady system samples: one row per second, averages and peaks
CREATE TABLE system_sample (
  t_ms INTEGER PRIMARY KEY,
  cpu_pm INTEGER, cpu_max_core_pm INTEGER, cpu_utility_pm INTEGER,
  perf_pm INTEGER, dpc_pm INTEGER, interrupt_pm INTEGER,
  queue_len INTEGER, ctx_switches INTEGER,
  mem_avail_kib INTEGER, commit_kib INTEGER, commit_limit_kib INTEGER, hard_faults INTEGER,
  disk_idle_pm INTEGER, disk_latency_us INTEGER, disk_bytes INTEGER,
  net_up_bytes INTEGER, net_down_bytes INTEGER,
  gpu_3d_pm INTEGER, gpu_decode_pm INTEGER,
  tick_late_max_us INTEGER
);

-- Steady process samples: selected processes only. process_id = 0 is the "other" total
CREATE TABLE process_sample (
  t_ms INTEGER NOT NULL,
  process_id INTEGER NOT NULL,
  cpu_pm INTEGER, working_set_kib INTEGER, private_kib INTEGER,
  io_bytes INTEGER, faults INTEGER, threads INTEGER, handles INTEGER,
  gpu_3d_pm INTEGER, gpu_decode_pm INTEGER, gpu_dedicated_kib INTEGER,
  net_up_bytes INTEGER, net_down_bytes INTEGER,
  PRIMARY KEY (t_ms, process_id)
) WITHOUT ROWID;

-- Network aggregation (administrator mode): 1 s buckets
CREATE TABLE net_bucket (
  t_ms INTEGER NOT NULL,
  process_id INTEGER NOT NULL,
  proto TEXT NOT NULL,          -- tcp | udp
  port_class TEXT NOT NULL,     -- 443 | 80 | other
  up_bytes INTEGER, down_bytes INTEGER, packets INTEGER,
  peers INTEGER,                -- distinct remote endpoints, counted through hashes
  PRIMARY KEY (t_ms, process_id, proto, port_class)
) WITHOUT ROWID;

-- Incidents
CREATE TABLE incident (
  id INTEGER PRIMARY KEY,
  kind TEXT NOT NULL,           -- auto | mark | external
  started_ms INTEGER NOT NULL, ended_ms INTEGER,
  capture_from_ms INTEGER, capture_to_ms INTEGER,
  signals TEXT NOT NULL,        -- JSON: [{rule, max, duration_ms}]
  suspects TEXT,                -- JSON: [{category, process_ids, delta, sentence}]
  note TEXT
);

-- Incident captures: high-resolution system samples (250 ms) and full process tables (500 ms)
-- One row per point in time. data is one compressed block holding every value at that point
CREATE TABLE capture_system (t_ms INTEGER PRIMARY KEY, data BLOB NOT NULL);
CREATE TABLE capture_process (t_ms INTEGER PRIMARY KEY, data BLOB NOT NULL);

-- Tool events: interval changes, source failures, rights, sleep and resume, clock anchors
CREATE TABLE tool_event (t_ms INTEGER NOT NULL, kind TEXT NOT NULL, detail TEXT);
```

- `_pm` is per mille (tenths of a percent), `_kib` is KiB, and `_us` is microseconds.
- `data` in the capture tables is an integer array in a fixed order, compressed with `lz4_flex`. `capture_process` puts every process at one point in time (`process_id` and its values) into one block, because compressing a small block per process gains almost nothing. The array order is stored in `capture_layout` in `meta`, so adding items does not change the tables.
- Primary keys start with time to match the report's queries (time ranges and per-incident captures).

## Size budget

| Part | Estimate | Basis |
| --- | --- | --- |
| Steady system samples | About 0.5 MB per hour | One row per second, about 25 values |
| Steady process samples | About 8 MB per hour | About 30 rows per second |
| Incident capture | About 6 MB raw and 2 MB compressed per incident (at most about 3.5 minutes) | 840 system points, 300 processes × 420 points. The compression ratio is an estimate |
| Network aggregation | About 2 MB per hour | Administrator mode, proportional to active processes |

- The target is under 20 MB per hour of steady recording ([N3 in the non-functional requirements](overview.md#non-functional-requirements)).
- The default maximum file size is 1 GB. Beyond it, the oldest steady samples are thinned to five-second spacing first. Incidents are never deleted.
- Phase 1 measures real sizes and replaces these estimates.

## Privacy boundary

The tool exists to receive recordings from other people's PCs, so it first defines what it never collects.

| Never collected | Kept instead |
| --- | --- |
| Window titles | Hung state and process key |
| Full command lines | `--type` and `--utility-sub-type` values of Chrome-family processes |
| Full executable paths (which contain user names) | Executable name |
| Raw remote IP addresses and ports | Port class (443, 80, other) and an endpoint count from hashes salted per recording |
| File names, document contents, the screen | Nothing |
| Computer name and user name | Nothing |
| Stream URLs and channel names (including in external logs) | Only times and state values are read from external logs |

- `meta` keeps only the hardware model (CPU and GPU names, total memory) and the OS build. They are needed to interpret causes and do not identify a person.
- Recordings are never sent off the PC automatically.
- The live screen's first-run notice and the end of the report show "what is and is not collected" in the same words.
- Tests use synthetic values only. Real recordings and personal PC information are never committed to the repository.

## Sharing

- From a friend, receive one `.lagtrace` file. The receiver can rebuild the report.
- The `.html` report is a single file that contains the recording data, so it can be viewed on its own. Detailed reanalysis still needs the `.lagtrace` file.
- If the file exceeds a messenger's attachment limit, `lag-tracker export --compact <file>` makes a copy with steady samples reduced to five-second spacing. Incident captures stay intact.
