# lag-tracker

A Windows tool written in Rust that records which processes were eating CPU, memory, GPU, and network at the moment your PC lagged or froze.

It records lightly all the time, and when a freeze is detected or you press a hotkey, it saves the preceding two minutes in detail.

> Design stage. There is no executable yet.

## What it will do

- Record system and per-process resource usage every second, and keep the last two minutes in memory at 250 ms resolution
- Detect CPU saturation, memory pressure, disk waits, GPU saturation, driver latency, and whole-system stalls automatically
- Save the preceding two minutes when you press `Ctrl+Alt+Shift+L` during a freeze
- Tell Chrome processes apart by role: browser, GPU, renderer, network service
- In administrator mode, attribute network traffic to processes and flag uploads that look like P2P
- Build a single-file HTML report from one recording, and overlay browser playback diagnostics on the same timeline

## Usage (planned)

1. Unzip the package and double-click `lag-tracker.exe`.
2. Press `Ctrl+Alt+Shift+L` every time the PC freezes.
3. When you are done, press `Q` in the window, then open the report or send the `.lagtrace` file it shows.

The screen and report follow the Windows display language (English or Korean).

## What it never collects

Window titles, full command lines, paths that contain user names, raw remote IP addresses, file or document contents, and screen contents. Nothing is sent off the machine automatically. See the [privacy boundary](docs/design/storage.md#privacy-boundary) for details.

## Documentation

- [Document index](docs/README.md)
- [Agent guidelines](AGENTS.md)

## Supported systems

Windows 10 1809 or later, Windows 11, x64

## License

Not decided yet. It will be chosen before the first implementation.
