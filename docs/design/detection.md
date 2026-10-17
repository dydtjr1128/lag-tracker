# Detection and Likely Causes

This document defines when something becomes an incident, what is saved, and how likely causes are chosen. Thresholds are initial values set before measurement. Adjust them with real recordings, and record each change here with its evidence.

## Principles

- Every verdict comes from **explainable rules**. No statistical models or machine learning.
- Results are shown as "likely causes", together with a sentence naming the numbers that crossed their thresholds.
- Happening at the same time does not prove causation. Report wording follows this principle too ([Copy principles](ui-ux.md#copy-principles)).

## Incident model

| Field | Meaning |
| --- | --- |
| Kind | `auto` (automatic rule), `mark` (user mark), `external` (imported from an external log) |
| Start and end | When a rule first became true and when it was last true. A mark has a single time |
| Signals | The rules that crossed their thresholds and the peak of each |
| Capture span | The saved high-resolution span: from 120 s before the start until 30 s after the end or 60 s after the start, whichever comes first |
| Likely causes | Resource category, the top three processes, and an evidence sentence |
| Note | Optional short text the user enters when marking |

## Ring buffer and capture

1. The sampler keeps the last 120 seconds in the ring buffer: system metrics at 250 ms, the process table at 500 ms, and GPU and network at 1 s resolution.
2. When an incident starts, the whole ring buffer is saved and high-resolution saving continues. When the incident ends, saving continues for 30 more seconds.
3. **High-resolution capture for one incident stops 60 seconds after it starts.** Saturation can last for hours during games or encoding, and one incident must not consume the whole capture budget. After that, only steady samples (1 s) are kept and only the incident's end time is updated. The report adds a "sustained" badge to incidents that lasted more than 30 seconds.
4. When a new incident starts during a capture, the same capture span is extended. The same data is never saved twice.
5. Captured spans are limited to 20 minutes per hour. Beyond that, only the incident summary is kept, and the report says so. User marks are always captured regardless of this limit.

## Automatic rules

| Rule | Condition (initial value) | Category | Phase |
| --- | --- | --- | --- |
| CPU saturation | Total CPU at 90% or more for 3 s or longer | CPU | 1 |
| Pinned core (informational) | One process uses at least one logical core's worth (100 ÷ core count %) for 10 s or longer | CPU | 1 |
| Memory pressure | Available memory under 10% and hard faults at 500 pages/s or more | Memory | 1 |
| Disk saturation | Disk idle under 10% and average latency of 50 ms or more for 3 s or longer | Disk | 1 |
| Hitch | The sampler tick wakes 500 ms or more later than scheduled | System stall | 1 |
| Driver latency | DPC plus interrupt time at 5% or more for 2 s or longer | Driver | 2 |
| Clock reduction | Total CPU at 50% or more while the clock ratio is under 70% for 10 s or longer | Power or heat | 2 |
| GPU saturation | Any process's `3D` or `VideoDecode` engine at 95% or more for 3 s or longer | GPU | 2 |
| Hung window | A visible window stays hung for 2 s or longer | Hung window | 2 |
| P2P suspicion | See [P2P suspicion](#p2p-suspicion) | Network | 3 |
| New process | A new process uses 5% CPU or more within 10 s of starting | Process | 3 |

- If the same rule becomes true again within 10 seconds of ending, the two are joined into one incident.
- When different rules overlap, one incident carries several signals.
- **Informational** rules never create an incident on their own, because they are always true for single-threaded games, encoders, and renderers. They attach as a signal to another incident, and otherwise only leave a "keeps one core busy" note on the report's process view.

### Hitch detection

The sampler wakes every 250 ms on a high-resolution waitable timer. Waking much later than scheduled means the thread could not run during that time.

- Lateness caused by the tool's own long processing is excluded. Only the **delay in waking from the wait** counts.
- Resume from sleep is filtered out using power events and the time difference.
- A hitch may happen while no resource crosses its threshold. Such incidents are labeled "no trace in other metrics", which is a reason to suspect drivers or hardware.
- The tool's process priority stays normal. Raising it could disturb other programs and would make the lateness measurement less representative of what the user experiences.
- Windows power throttling, however, is turned off. Otherwise a minimized tool could be throttled, wake late, and miscount that as a system hitch ([Process settings](architecture.md#process-settings)).

## User marks

| Input | Behavior |
| --- | --- |
| `Space` on the live screen | Marks immediately. A short note can follow (Enter skips it) |
| Global hotkey (default `Ctrl+Alt+Shift+L`) | Marks immediately, without a note. Works while another program is in use |
| `lag-tracker mark [--note text]` | Marks the running recording. For scripts and other tools |

- A key pressed while the PC is frozen is processed late. Saving the 120 seconds before the mark makes sure the actual freeze is included.
- The mark time is when the key event reached the tool. The report adds a note that the mark was pressed after the freeze, so the freeze itself happened earlier.
- Several presses within one second count as one mark.

## Choosing likely causes

Each incident gets likely causes in this order.

1. **Resource category**: collect the categories of the rules that crossed their thresholds during the incident. If there are several, sort them by how far they exceeded their thresholds.
2. **Baseline**: take each process's median from 60 to 10 seconds before the incident started.
3. **Contribution**: for the relevant resource, compute `incident average - baseline` per process. The larger the increase, the higher the rank. A process that **suddenly increased** ranks above one that was always heavy.
4. **Grouping**: Chrome-family processes also get a total per role, for example "Chrome renderers (3) +41 pp".
5. **Evidence sentence**: one sentence with the resource, duration, peak, and top contributions.

Example evidence sentences:

- `Total CPU at 97% for 4.2 s. Increased the most: Chrome renderers (2) +38 pp, Chrome GPU +12 pp`
- `GPU video decode engine at 98% for 6 s. Highest user: Chrome GPU`
- `Sampler woke 1.3 s late. CPU, memory, disk, and GPU were below their thresholds at that time`

## P2P suspicion

How P2P delivery such as Chzzk grid actually communicates, and from which process, has not been confirmed. Detection therefore relies on **traffic shape**, not process names.

| Condition (initial value, 10 s moving window) | Reason |
| --- | --- |
| Average upload of 1 Mbps or more | Upload is tiny when only watching |
| UDP to a remote port other than 443 makes up 50% or more of all upload | HTTP/3 from a CDN uses UDP port 443 |
| 5 or more distinct remote endpoints | P2P exchanges data with many peers at once |

- Meeting all three conditions is labeled "P2P suspected", and meeting two is labeled "heavy upload".
- Standard mode only sees total upload without process attribution. It stops at "heavy total upload" and suggests administrator mode.
- The thresholds are tuned by comparing two recordings with grid bypass on and off ([Roadmap phase 3](roadmap.md#phase-3-network-and-p2p-suspicion)).

## External log overlay (phase 4)

When external logs are passed while building a report, they are overlaid on the same timeline. Their events also become `external` incidents and get likely causes in the same way.

| Input | How it is recognized | What is imported | Time base |
| --- | --- | --- | --- |
| Chzzk Light diagnostics | `format` is `chzzk-light-playback-diagnostics` | `waiting`, `rebuffer`, `long-task`, `timer-delay`, and `progress-stall` events, and the dropped-frame ratio from `sample` | Epoch milliseconds |
| media-internals log | `events[]` and a `kMediaLogCreated` event | `BUFFERING_HAVE_NOTHING` spans for video, audio, and pipeline, with their reasons | `created` (UTC) plus relative seconds |

- External logs are read only while building the report and are not copied into the recording. Identifying fields such as channel names are never read.
- PC clocks can differ, so the guidance is to overlay only logs made on the same PC. When an offset is suspected, `--offset` corrects it.
- If an external log does not overlap the recording, the report says so.

## Validation

- **Synthetic tests**: the rules and contribution calculation in `lag-core` are unit tested with hand-made time series, covering threshold boundaries (just below and just above), joining, and the capture limits.
- **Load generator**: a test executable saturates N CPU threads, allocates large amounts of memory, or writes to disk continuously for a set time. The test checks that the incident appears and that the first likely cause is correct.
- **Real conditions**: record under conditions that reproduce the problem, such as two 1080p60 videos playing at once, and overlay the browser diagnostics to check that times line up.
