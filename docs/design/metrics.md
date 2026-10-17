# Metrics

This document defines what is collected, from where, how often, and with which rights. When a metric is added or changed, update this document and the [privacy boundary](storage.md#privacy-boundary) together.

## Common rules

- PDH counters are added with `PdhAddEnglishCounterW` using English names. Counter names are localized on non-English Windows (Korean, for example), so adding the English path with the localized API fails there.
- Per-second (`/sec`) and ratio counters need two collections before the first value appears. The first tick after start is discarded.
- Wildcard (`(*)`) instances are expanded on every collection, because instances change as processes start and exit.
- Ratios are stored as integers in tenths of a percent (per mille), and memory as integers in KiB.
- "Phase" refers to the implementation phases in the [Roadmap](roadmap.md).

## System metrics (PDH)

| Item | Counter | Interval | Phase | Meaning |
| --- | --- | --- | --- | --- |
| Per-core and total CPU | `\Processor Information(*)\% Processor Time` | 250 ms | 1 | Total of 90% or more that persists is a CPU saturation candidate |
| CPU as Task Manager shows it | `\Processor Information(_Total)\% Processor Utility` | 250 ms | 1 | For comparison with Task Manager. Can exceed 100% with turbo boost |
| Clock ratio | `\Processor Information(_Total)\% Processor Performance` | 250 ms | 2 | Under 70% while busy is a heat or power saving candidate |
| DPC time | `\Processor Information(_Total)\% DPC Time` | 250 ms | 2 | 5% or more is a driver latency candidate |
| Interrupt time | `\Processor Information(_Total)\% Interrupt Time` | 250 ms | 2 | Read together with DPC time |
| Run queue | `\System\Processor Queue Length` | 250 ms | 1 | More than twice the logical core count means threads are waiting in line |
| Context switches | `\System\Context Switches/sec` | 250 ms | 1 | Reference for spikes |
| Available memory | `\Memory\Available MBytes` | 250 ms | 1 | Under 10% of total is a memory pressure candidate |
| Commit | `\Memory\Committed Bytes`, `\Memory\Commit Limit` | 1 s | 1 | Commit close to the limit means allocation failures or page file growth |
| Hard page faults | `\Memory\Pages Input/sec` | 250 ms | 1 | A spike means memory is being read back from disk |
| Disk idle and latency | `\PhysicalDisk(*)\% Idle Time`, `\PhysicalDisk(*)\Avg. Disk sec/Transfer` | 250 ms | 1 | Idle under 10% with latency of 50 ms or more is a disk saturation candidate |
| Disk throughput | `\PhysicalDisk(*)\Disk Bytes/sec` | 1 s | 1 | Spots updates and antivirus scans |
| Total network | `\Network Interface(*)\Bytes Sent/sec`, `Bytes Received/sec` | 1 s | 1 | Shows total upload even in standard mode. Per-process attribution needs administrator mode |
| Temperature (when available) | `\Thermal Zone Information(*)\Temperature` | 5 s | 2 | An ACPI value from the motherboard that may differ from the real CPU temperature. Reference only |

## Process metrics (NtQuerySystemInformation)

One `SystemProcessInformation` call reads every process. It runs every 500 ms.

| Item | Source field | Phase | Calculation and meaning |
| --- | --- | --- | --- |
| CPU usage | `KernelTime + UserTime` | 1 | Difference from the previous collection ÷ (elapsed time × logical cores). A share of total CPU, like Task Manager |
| Working set | `WorkingSetSize` | 1 | Memory actually resident |
| Private bytes | `PrivatePageCount` | 1 | Commit used only by that process. Used to track leaks |
| Page faults | `PageFaultCount` | 1 | Soft and hard faults combined. This API cannot separate hard faults per process |
| I/O throughput | `ReadTransferCount`, `WriteTransferCount`, `OtherTransferCount` | 1 | May include device I/O other than files |
| Threads and handles | `NumberOfThreads`, `HandleCount` | 1 | Reference for spikes and leaks |
| Parent process | `InheritedFromUniqueProcessId` | 1 | Groups Chrome child processes under the browser |
| Creation time | `CreateTime` | 1 | Key that tells reused PIDs apart |

### Chrome roles

For Chrome-family executables, only the `--type=` and `--utility-sub-type=` values are read from the command line and turned into a role.

| Command line value | Role shown |
| --- | --- |
| No `--type` | Browser |
| `--type=gpu-process` | GPU |
| `--type=renderer` | Renderer |
| `--type=utility`, `--utility-sub-type=network.mojom.NetworkService` | Network service |
| `--type=utility`, `--utility-sub-type=audio.mojom.AudioService` | Audio |
| `--type=utility`, any other value | Utility |
| `--type=crashpad-handler` | Crash reporter |
| Could not read | Unknown |

The default executables are `chrome.exe`, `msedge.exe`, and `whale.exe`. The settings file can add more.

### Processes kept in steady samples

Storing every process every second would make the file large. Steady samples keep only processes that match one of the rules below, and fold the rest into one "other" row so that totals still match the system figures.

- The top 10 by CPU in that second
- The top 10 by private bytes (reselected every 10 seconds)
- Watched processes: every Chrome-family process and any name in the settings file
- Any process with CPU of 1% or more, any GPU engine at 1% or more, or network of 100 KB/s or more

When an incident occurs, **every process** in the ring buffer is saved as is.

## GPU metrics (PDH, phase 2)

| Item | Counter | Meaning |
| --- | --- | --- |
| Per-process engine usage | `\GPU Engine(*)\Utilization Percentage` | The process and engine type come from `pid_<PID>_..._engtype_<type>` in the instance name |
| Per-process GPU memory | `\GPU Process Memory(*)\Dedicated Usage`, `Shared Usage` | Dedicated and shared memory |
| Adapter GPU memory | `\GPU Adapter Memory(*)\Dedicated Usage` | Graphics memory pressure |

- Engine types are grouped as `3D`, `VideoDecode`, `VideoEncode`, `Copy`, and `Compute`. For video stutter analysis, `3D` and `VideoDecode` matter most.
- When several engines share a type, the per-process value for that type is the maximum. A sum can exceed 100% and blur saturation.
- These counters have many instances and can be expensive. Phase 1 measurements decide the interval ([Tick plan](architecture.md#tick-plan)).

## Hung windows (phase 2)

- Every second, the tool walks visible top-level windows and records the process of any window for which `IsHungAppWindow` is true.
- Window titles are never read. Only the process key and the start and end of the hung state are kept.

## Network (administrator mode, phase 3)

TCP and UDP send and receive events from the ETW `Microsoft-Windows-Kernel-Network` provider are aggregated into one-second buckets.

| Aggregation key | Values |
| --- | --- |
| Process key, protocol (TCP or UDP), direction (up or down), remote port class (443, 80, other) | Total bytes, packet count |
| Process key, protocol | Number of distinct remote endpoints, largest upload to a single endpoint |

- Raw remote IP addresses and ports are never stored. They are hashed with a random value created for each recording, only to count how many distinct endpoints there are.
- Chrome's ordinary HTTP traffic happens in the network service process. Which process owns WebRTC P2P sockets is verified in phase 3.
- HTTP/3 (QUIC) from a CDN uses UDP port 443. P2P suspicion therefore looks at "UDP to a remote port other than 443" ([P2P suspicion](detection.md#p2p-suspicion)).

## Process start and exit (phase 3)

- In administrator mode the tool receives start and exit events from the ETW `Microsoft-Windows-Kernel-Process` provider, which also catches short-lived processes.
- In standard mode it infers starts and exits by comparing process tables. Processes that live less than 500 ms can be missed.

## The tool itself

The tool stores its own overhead in the same file, so whoever reads the recording can judge its influence.

| Item | Interval |
| --- | --- |
| Own CPU and working set | 1 s |
| Tick processing time per source | Every tick, peak stored per second |
| Tick lateness (how late it woke versus schedule) | Every tick, peak stored per second |
| Writer queue length and thinned samples | 1 s |
| Interval change events | When they occur |

## Reading the numbers

| What you see | Suspect first |
| --- | --- |
| Total CPU 90% or more, run queue at least twice the core count | CPU saturation. Check the top processes |
| CPU has headroom but the clock ratio is low | Power saving plan or heat |
| Available memory under 10% and hard faults spiking | Memory pressure. Check the top processes by private bytes |
| Disk idle under 10% and latency of 50 ms or more | Updates, antivirus scans, the page file |
| High DPC or interrupt time | A driver problem. Find the driver with WPR/WPA or LatencyMon |
| GPU `VideoDecode` or `3D` at 95% or more | Video decode or compositing saturation. Check the number of simultaneous videos and their quality |
| Large tick lateness while other metrics are quiet | A brief whole-system stall. Suspect drivers, power state transitions, or hardware |
