# Overview and Requirements

This document defines what lag-tracker solves, who uses it and how, and what it must satisfy to be done. How it is built is in [Architecture](architecture.md).

## Background

- Video playback on a Windows PC sometimes stutters or the screen freezes briefly, and nothing records which program was using resources at that moment.
- Task Manager only shows the present. A freeze passes within seconds, so everything looks normal by the time Task Manager opens.
- Browser-side diagnostics (for example the Chzzk Light playback diagnostics or `chrome://media-internals`) can show that frames were dropped while data was still buffered. They cannot show what was consuming CPU or GPU across the whole PC at that time.
- In playback diagnostics from another PC, stutters in two windows got worse and recovered at the same moments. That pointed to a shared resource being saturated, but browser logs alone could not identify the responsible process or tell whether grid (P2P) delivery was active.

## Questions to answer

1. Which **processes** used how much CPU, memory, GPU, disk, and network around the moment of a freeze?
2. Which **resource** is the likely cause: CPU saturation, memory pressure and page faults, disk waits, GPU engine saturation, driver latency (DPC and interrupts), or clock reduction from heat or power saving?
3. Inside the browser, which **Chrome process role** was responsible: browser, GPU, renderer, network service, or audio?
4. Was there an **upload that looks like P2P** (such as grid delivery) at that moment?
5. Can a recording from **another PC arrive as one file** and be analyzed the same way?

## Goals

- Record lightly all the time, and save the preceding two minutes in detail when a freeze is detected or the user marks one.
- Explain likely causes with one recording file and one report. Present every verdict as a "likely cause" together with the numbers behind it.
- Let a user with no development background double-click the executable, press a hotkey when the PC freezes, and send a file when done.
- Set an overhead budget so the tool never becomes the cause of lag, and store its own overhead in the recording.

## Non-goals

- Fixing causes automatically or terminating processes.
- Installing a kernel driver. Per-driver DPC latency analysis is left to specialist tools such as LatencyMon and WPR/WPA.
- Attributing usage to individual Chrome **tabs**. An external process has no public, reliable way to map renderers to tabs, so attribution stops at process roles.
- Supporting macOS or Linux.
- Sending recordings anywhere. Users share files themselves.
- Starting with Windows or running as an always-on service.

## Users

| User | Situation | Needs |
| --- | --- | --- |
| The developer | Digs into causes on their own PC | Detailed numbers, external log overlay, command-line options |
| A friend or reporter | Records on another PC on the developer's behalf | No installation, guidance in their language, one-button-level operation, a clear statement of what is collected |

## Key scenarios

1. **Tracking intermittent freezes**: run, use the PC as usual, press the hotkey on a freeze, quit hours later, then check the top processes around each mark in the report.
2. **Recording on another PC**: unzip and run, record, then send the single file named on the exit screen through a messenger.
3. **Comparing conditions**: record twice while changing only one condition (grid bypass on or off, playback quality, number of tabs), then compare resource usage and incident counts.
4. **Overlaying external logs**: put Chzzk Light diagnostics and media-internals logs on the same timeline to line up stall times with resource saturation.

## Functional requirements

| ID | Requirement | Phase |
| --- | --- | --- |
| F1 | Record system metrics (per-core CPU, memory, disk) every second | 1 |
| F2 | Record per-process CPU, memory, I/O, and thread count, and identify Chrome process roles | 1 |
| F3 | Keep the last two minutes in memory at high resolution and save them when an incident occurs | 1 |
| F4 | Mark freezes with a console key, a global hotkey, or a separate `mark` command | 1 |
| F5 | Detect whole-system stalls (hitches) from the tool's own timer lateness | 1 |
| F6 | Build an HTML report from a recording | 1 |
| F7 | Record per-process GPU engine usage (3D and video decode) and GPU memory | 2 |
| F8 | Detect DPC and interrupt time, clock ratio, and hung windows | 2 |
| F9 | Rule-based likely causes with an evidence sentence | 2 |
| F10 | Per-process network and P2P suspicion in administrator mode | 3 |
| F11 | Record process start and exit | 3 |
| F12 | Overlay Chzzk Light diagnostics and media-internals logs | 4 |
| F13 | Comparison report for two recordings | 4 |

## Non-functional requirements

| ID | Requirement | Target |
| --- | --- | --- |
| N1 | Tool CPU overhead | Under 0.5% of total CPU averaged over one hour, under 2% in any one-second window |
| N2 | Tool memory | Working set under 80 MB |
| N3 | Disk usage | Under 20 MB per hour of steady recording |
| N4 | Distribution | Single executable with no installer and no extra runtime |
| N5 | Permissions | Core features work as a standard user. Administrator rights are needed only for network detail |
| N6 | Durability | Closing the console window or shutting down Windows keeps data up to the last five seconds |
| N7 | Privacy | Never store window titles, full command lines, paths containing user names, or raw remote IP addresses |
| N8 | Supported OS | Windows 10 1809 or later, Windows 11, x64 |
| N9 | Language | Screen and report in English and Korean, following the Windows display language ([UI/UX](ui-ux.md#language)) |

These numbers are targets set before the first measurement. Phase 1 measures them and turns them into acceptance criteria in the [Roadmap](roadmap.md).

## Success criteria

- A three-second CPU saturation produced by a load generator becomes an incident, and the load generator ranks first in that incident's detail in the report.
- The two minutes before a user mark are kept at 250 ms resolution.
- The same report can be rebuilt from a single recording file received from another PC.
- A friend can go from recording to sending the file using only the on-screen guidance, without reading any document.

## Terms

| Term | Meaning |
| --- | --- |
| Recording | One collection session from start to exit, stored in one recording file (`.lagtrace`) |
| Steady sample | System metrics and selected process metrics stored every second |
| Ring buffer | High-resolution samples of the last 120 seconds, kept only in memory |
| Incident | A time span created by an automatic rule or a user mark, saved together with the ring buffer contents |
| Mark | A moment the user reported as "it just froze" |
| Hitch | A moment when the tool's periodic timer woke up very late, a sign that the whole system stalled briefly |
| Likely cause | A resource that crossed its threshold during an incident, and the process that increased it the most |
