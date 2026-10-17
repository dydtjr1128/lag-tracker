# Roadmap

This document defines the implementation phases, their acceptance criteria and validation, risks, and decisions still open. Each phase is cut so that it delivers something useful on its own.

## Phases at a glance

| Phase | Result | Question it answers |
| --- | --- | --- |
| 0 | Repository skeleton and check commands | None (preparation) |
| 1 | Steady CPU, memory, and disk recording, incident capture, marks, basic report | Which processes used CPU and memory at the moment of a freeze? |
| 2 | GPU, DPC, clock, hung windows, likely cause sentences | Which resource is the likely cause? |
| 3 | Administrator-mode network, P2P suspicion, process start and exit | Was there P2P upload such as grid delivery? |
| 4 | External log overlay, comparison of two recordings | Do browser stutters line up with resource saturation, and does changing a condition help? |
| 5 | Tray mode (optional) | Convenience for long recordings |

## Phase 0: repository skeleton

- Create the Cargo workspace and its three crates (`lag-core`, `lag-win`, `lag-tracker`) ([Crate layout](architecture.md#crate-layout)).
- Set `rust-version` (MSRV) and `rust-toolchain.toml`.
- Fix the check commands and fill in the command table in [AGENTS.md](../../AGENTS.md): `cargo fmt --all --check`, `cargo clippy --workspace --all-targets -- -D warnings`, `cargo test --workspace`.
- Choose a license and add `LICENSE` ([Open questions](#open-questions)).

Acceptance criteria:

- [ ] An empty executable builds and the three checks above pass
- [ ] No `Not set` remains in the command table of AGENTS.md

## Phase 1: steady recording and incident capture (MVP)

Scope:

- System PDH metrics (CPU, memory, disk, total network), the process table, and Chrome roles ([Metrics](metrics.md))
- Ring buffer, hitch detection, CPU, memory, and disk rules, and all three ways to mark ([Detection](detection.md))
- SQLite recording file ([Storage](storage.md)) and shutdown signal handling
- Live screen, first-run notice, exit summary, English and Korean strings ([UI/UX](ui-ux.md))
- Report: header, incident list, timeline (CPU, memory, disk, hitches), process table in incident detail
- Overhead measurement and degradation

Acceptance criteria:

- [ ] Saturating the CPU for three seconds with a load generator creates exactly one incident, and the load generator ranks first in its detail
- [ ] For a mark made with the global hotkey, the preceding 120 seconds are kept at 250 ms resolution
- [ ] In a one-hour recording, the tool averages under 0.5% CPU, peaks under 2% in any second, and keeps its working set under 80 MB
- [ ] A one-hour steady recording file is under 20 MB
- [ ] Closing the console window during recording keeps data up to the last five seconds, and a report can still be built
- [ ] Every PDH counter registers on Korean Windows
- [ ] A one-hour recording without load and with the console window minimized has zero false hitches ([Process settings](architecture.md#process-settings))
- [ ] During CPU saturation lasting over 30 minutes, high-resolution capture for the single incident stops at 60 seconds
- [ ] The screen and report appear in Korean on Korean Windows and in English elsewhere

Validation:

- Unit tests: rule boundaries, joining, ring buffer overwrite, CPU ratio calculation
- Load generator scenarios: CPU saturation, large memory allocation, continuous disk writes
- Manual: record while playing a 1080p60 video and compare with Task Manager

## Phase 2: GPU and likely causes

Scope:

- GPU engines and GPU memory, DPC and interrupt time, clock ratio, temperature (when available), hung windows
- Driver latency, clock reduction, GPU saturation, and hung window rules
- Contribution calculation and evidence sentences, summary cards and the GPU row in the report

Acceptance criteria:

- [ ] Measure the cost of GPU engine collection, fix its interval, and record the result in the [Tick plan](architecture.md#tick-plan)
- [ ] With two videos playing at once, `VideoDecode` and `3D` usage is attributed to the Chrome GPU process
- [ ] Switching the power plan to power saver and adding load creates a clock reduction incident
- [ ] A deliberately frozen window (test program) is captured as a hung window incident

## Phase 3: network and P2P suspicion

Scope:

- Administrator mode, ETW network aggregation, process start and exit, the restart-as-administrator flow
- P2P suspicion rule and the network row in the report

Acceptance criteria:

- [ ] In administrator mode, video streaming traffic is attributed to the Chrome network service process
- [ ] Verify which Chrome process owns WebRTC P2P sockets and record it in [Metrics](metrics.md#network-administrator-mode-phase-3)
- [ ] Record once with grid bypass on and once with it off, compare upload volume and endpoint counts, and fix the thresholds with that evidence
- [ ] A check confirms that no raw remote IP address or port appears anywhere in a recording file
- [ ] The next run cleans up an ETW session left behind by an abnormal exit
- [ ] A standard-user `lag-tracker mark` and second-launch screen reach an administrator recording through the named pipe. If not, adjust the pipe's security settings

## Phase 4: external logs and comparison

Scope:

- Import Chzzk Light diagnostics and media-internals logs
- Likely causes for external incidents and the external row in the report
- Comparison report for two recordings

Acceptance criteria:

- [ ] Test time alignment and the "no overlap" notice with synthetic diagnostics (fictional values)
- [ ] With two real logs from the same PC, the browser's stall start times and resource saturation can be compared on one screen
- [ ] A comparison shows incidents per hour, average CPU and GPU, and changes in top processes

## Phase 5: tray mode (optional)

Start only if phases 1 to 4 show a need in real use. The scope is [Tray mode in UI/UX](ui-ux.md#tray-mode-candidate-phase-5).

## Release

- Versions follow SemVer `0.x` in `Cargo.toml`. Completing phase 1 is `0.1.0`.
- GitHub Releases publish a zip with the executable and `HOW-TO-USE.txt`, plus SHA-256 hashes.
- Changes are described in release notes. A separate CHANGELOG file is added when it becomes necessary.
- CI and automated releases are added only on request.
- Before the first release, add `THIRD_PARTY_NOTICES.md` with the licenses of uPlot (embedded in reports) and of the crates linked into the executable, and include it in the zip.

## Risks and mitigations

| Risk | Impact | Mitigation |
| --- | --- | --- |
| GPU engine counters are expensive to collect | The tool causes lag | Measure and tune the interval, evaluate `D3DKMTQueryStatistics`, degrade |
| Performance counters are corrupted or disabled | Some metrics missing | Turn off per item and continue. `doctor` explains `lodctr /r` |
| Antivirus false positives | Execution blocked | Publish hashes, document how to report false positives, never request unnecessary rights |
| SmartScreen warning | A friend gives up before running it | Steps in `HOW-TO-USE.txt`, mentioned on its first line |
| Hotkey conflicts | Cannot mark | Report the registration failure, fall back to `Space` and `mark`, allow `--hotkey` |
| PID reuse | Mistaking one process for another | `(PID, creation time)` key |
| System clock changes and sleep | Misaligned timeline, false hitches | Monotonic clock, new anchors, separate sleep spans |
| Taskbar flashing does not work under Windows Terminal | Weak hotkey feedback | Verify in phase 1, fall back to the window title change only |
| Wrong likely causes | Wrong conclusions | "Likely cause" wording, visible evidence numbers, a record of threshold changes |

## Open questions

| Question | Default proposal | Decide by |
| --- | --- | --- |
| License | MIT | Phase 0 |
| Default global hotkey | `Ctrl+Alt+Shift+L`, with `Pause` as the single-key alternative | After phase 1 real use |
| Build the report automatically on exit | Build automatically, open only when the user chooses | Phase 1 |
| Default maximum recording duration | No limit, 1 GB file cap | After phase 1 measurements |
| Capture limit | 20 minutes of capture per hour | After phase 1 measurements |
| GPU collection path | PDH first, D3DKMT if PDH is too expensive | After phase 2 measurements |
| Whether to build tray mode | Decide after phase 4 | After phase 4 |
