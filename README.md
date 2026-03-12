![Release](https://img.shields.io/github/v/release/NoCoderRandom/killall)
![Downloads](https://img.shields.io/github/downloads/NoCoderRandom/killall/total)
![License](https://img.shields.io/github/license/NoCoderRandom/killall)

# killall — Windows 11 Intelligent Process Termination Tool

A modern, safe, Windows-native equivalent of UNIX `killall` with three-tier safety protection, process tree awareness, and specialized subcommands for GPU, network, hung apps, LLM, and game processes.

## Installation

### Prerequisites
- Windows 10 1809+ / Windows 11
- .NET 8 SDK (for building from source)

### Build from source

```powershell
git clone <repo-url>
cd killall
dotnet publish -c Release -r win-x64 --self-contained true
```

The output binary is at `bin\Release\net8.0-windows\win-x64\publish\killall.exe` (34 MB self-contained, no runtime dependency).

### Add to PATH

Copy `killall.exe` to a directory on your PATH, or add the publish directory:

```powershell
# Option 1: Copy to a bin directory
copy bin\Release\net8.0-windows\win-x64\publish\killall.exe C:\Tools\killall.exe

# Option 2: Add to user PATH
[Environment]::SetEnvironmentVariable("Path", $env:Path + ";C:\Tools", "User")
```

## Quick Start

```powershell
# Kill all instances of Notepad
killall notepad

# Kill Chrome and its entire process tree
killall chrome --tree

# Kill all hung/frozen applications
killall hung

# Kill processes using more than 2 GB RAM
killall ramhog 2048

# Kill top 3 CPU hogs above 90%
killall cpuhog 90 --top 3

# Preview what would be killed (dry run)
killall game --dry-run
```

## Usage

```
killall <pattern> [options]
killall <subcommand> [options]
```

### Pattern Matching

| Pattern         | Mode       | Example                         |
|-----------------|------------|---------------------------------|
| `notepad`       | Exact      | Matches `notepad` (case-insensitive) |
| `note`          | Partial    | Falls back to substring if no exact match |
| `note*`         | Glob       | Wildcard matching               |
| `/fire.*fox/`   | Regex      | Full regex enclosed in slashes  |

### Global Options

| Flag              | Short | Description                          |
|-------------------|-------|--------------------------------------|
| `--tree`          | `-t`  | Kill entire process tree (children first) |
| `--force`         | `-f`  | Skip confirmation prompt             |
| `--dry-run`       | `-n`  | Show what would be killed            |
| `--help`          | `-h`  | Show help                            |

### Advanced Filters

These flags can be used standalone or combined with a name pattern and all global options (`--tree`, `--force`, `--dry-run`). All filters respect the three-tier safety model.

| Flag                        | Description                                          |
|-----------------------------|------------------------------------------------------|
| `--cmdline <pattern>`       | Kill processes whose command line matches a substring or `/regex/` |
| `--module <dllname>`        | Kill processes that have loaded the specified DLL     |
| `--port <N>` or `<N-M>`    | Kill processes owning the given TCP/UDP port or range |
| `--window <title>`          | Kill processes owning a window matching title substring or `/regex/` |
| `--parent <pid or name>`    | Kill all direct children of the specified parent      |

#### Examples

```powershell
# Kill any process launched with "--model" in its arguments
killall --cmdline "--model"

# Kill Python http.server instances (regex match)
killall --cmdline "/http\.server/"

# Kill all processes that loaded Direct3D 12
killall --module d3d12.dll

# Kill processes using CUDA via PyTorch
killall --module torch_cuda.dll

# Kill whatever is listening on port 8080
killall --port 8080

# Kill all processes bound to ports 5000–6000
killall --port 5000-6000

# Kill windows with "Crash Reporter" in the title
killall --window "Crash Reporter"

# Kill Chrome Incognito windows (regex)
killall --window "/Chrome.*Incognito/"

# Kill all child processes of explorer.exe
killall --parent explorer

# Kill children of PID 1234 and their entire subtrees
killall --parent 1234 --tree

# Combine: kill processes named "node" that are children of PID 5678
killall node --parent 5678
```

### Subcommands

| Command                       | Description                                   |
|-------------------------------|-----------------------------------------------|
| `killall restart <name>`      | Kill and restart a process                    |
| `killall gpu [--threshold N]` | Kill GPU-consuming processes (default >1%)    |
| `killall networkapps`         | Kill processes with active network connections|
| `killall hung`                | Kill all "Not Responding" applications        |
| `killall ramhog <MB>`         | Kill processes exceeding N MB RAM             |
| `killall cpuhog <pct>`        | Kill processes exceeding N% CPU               |
| `killall llm`                 | Kill all local LLM/AI inference processes     |
| `killall game`                | Kill all game processes and launchers         |

### Subcommand Options

| Option            | Applies to       | Description                         |
|-------------------|------------------|-------------------------------------|
| `--threshold N`   | gpu              | Minimum GPU utilization % (default 1) |
| `--top N`         | ramhog, cpuhog   | Kill only top N offenders           |
| `--sample N`      | cpuhog           | CPU sampling interval in seconds (default 1) |

## Safety Model

killall implements a strict three-tier protection system to prevent system crashes.

### Tier 1 — IMMORTAL (Hard Block)

**Never terminated under any circumstances.** Attempting to kill these processes will be silently skipped with a log message.

Includes: `System`, `csrss`, `lsass`, `winlogon`, `svchost`, `dwm`, `smss`, `wininit`, `services`, `fontdrvhost`, `MsMpEng`, `Registry`, `conhost`, `SecurityHealthService`, and others.

Killing these would cause an immediate BSOD or system crash.

### Tier 2 — AUTO-RESTART (Watchdog)

**Terminated, then immediately restarted.** These are essential Windows 11 UI processes.

Includes: `explorer`, `ShellExperienceHost`, `StartMenuExperienceHost`, `SearchHost`, `RuntimeBroker`, `Taskmgr`, `ApplicationFrameHost`, `Widgets`.

`explorer.exe` has special retry logic — up to 3 restart attempts with 2-second intervals and fallback instructions if all attempts fail.

### Tier 3 — ALLOWED (Normal Kill)

All other processes. Killed normally with user confirmation (unless `--force` is used).

## Advanced Filter Detection Methods

### Command-Line Filter (`--cmdline`)
- Reads the full command line of every process via WMI `Win32_Process.CommandLine`.
- Supports plain substring matching (case-insensitive) or full regex when enclosed in `/slashes/`.

### Module Filter (`--module`)
- Enumerates loaded modules via `Process.Modules` for each accessible process.
- Case-insensitive match on the module file name.
- Gracefully skips IMMORTAL-tier processes and handles 32-bit/64-bit access errors.

### Port Filter (`--port`)
- Reuses `GetExtendedTcpTable` / `GetExtendedUdpTable` (iphlpapi.dll P/Invoke).
- Supports single port (`8080`) or range (`5000-6000`).
- Scans both TCP and UDP, IPv4 and IPv6.

### Window Title Filter (`--window`)
- Enumerates all top-level windows via `EnumWindows` (user32.dll).
- Reads window titles via `GetWindowText`.
- Supports substring or `/regex/` matching.
- Maps window handles to owning PIDs via `GetWindowThreadProcessId`.

### Parent Filter (`--parent`)
- Accepts a numeric PID or a process name.
- If a name is given, resolves it to PID(s) first.
- Uses WMI `Win32_Process.ParentProcessId` to find all direct children.
- Combines naturally with `--tree` to kill entire subtrees.

## Detection Methods

### GPU Process Detection
- **Primary**: WMI GPU Engine performance counters (`Win32_PerfFormattedData_GPUPerformanceCounters_GPUEngine`). Parses PID from counter instance names and sums utilization across all GPU engines.
- **Fallback**: Loaded module detection — identifies processes that loaded GPU-related DLLs (`d3d11.dll`, `d3d12.dll`, `vulkan-1.dll`, `nvcuda.dll`, etc.).
- Supports NVIDIA, AMD, and Intel GPUs.

### Network Connection Detection
- P/Invoke to `iphlpapi.dll` using `GetExtendedTcpTable` and `GetExtendedUdpTable`.
- Enumerates all TCP connections (ESTABLISHED state) and UDP endpoints with owning PIDs.
- Supports IPv4 and IPv6.

### Hung Application Detection
- `IsHungAppWindow` (user32.dll) for visible windows.
- `SendMessageTimeout` with `SMTO_ABORTIFHUNG` flag as secondary detection.
- `Process.Responding` property as a tertiary check.
- Scans all visible windows via `EnumWindows`.

### LLM Process Detection
- Matches against 30+ known LLM process names (Ollama, llama.cpp, LM Studio, KoboldCpp, vLLM, ComfyUI, etc.).
- Scans command lines for patterns like `.gguf`, `transformers`, `torch.cuda`, `--model`, `--load-in-4bit`.
- Detects Python processes running LLM workloads.

### Game Process Detection
- Known game launchers: Steam, Epic, Battle.net, GOG Galaxy, Ubisoft Connect, EA Desktop, Riot.
- Executable path heuristics: processes running from `steamapps`, `Epic Games`, `GOG Games`, etc.
- Module detection: processes loading `UnityPlayer.dll`, `steam_api64.dll`, `XInput`, `dinput8.dll`, etc.
- Requires 2+ game-related modules for module-based detection to reduce false positives.

### CPU Usage Measurement
- Two-sample measurement using `Process.TotalProcessorTime`.
- Configurable sampling interval (default 1 second).
- Normalizes across all logical CPU cores.

## Process Tree Kill (`--tree`)

When `--tree` is specified:
1. A complete parent-child process map is built via WMI `Win32_Process`.
2. All descendants of each matched process are recursively collected using BFS.
3. Processes are killed bottom-up (children before parents).
4. Tier 1 processes in the tree are skipped. Tier 2 processes are killed and restarted.

## Exit Codes

| Code | Meaning                       |
|------|-------------------------------|
| 0    | All operations succeeded      |
| 1    | Partial failure (some denied) |
| 2    | All operations failed/refused |
| 3    | No matching processes found   |

## Administrator Privileges

killall works without elevation for user-owned processes. For full functionality (killing services, system-level processes), run from an elevated terminal. A warning is shown when running without Administrator privileges.

## Architecture

Single-file C# 12 / .NET 8 console application. No external dependencies beyond `System.Management` (WMI).

| Component            | Role                                              |
|----------------------|---------------------------------------------------|
| `SafetyService`      | Three-tier classification and restart logic        |
| `ProcessTreeService` | WMI-based process tree builder                     |
| `PatternMatcher`     | Multi-mode process name matching                   |
| `KillCommand`        | Core kill engine with safety evaluation            |
| `NativeMethods`      | P/Invoke declarations (user32, iphlpapi)           |
| 8 subcommand classes | Specialized detection and kill logic               |
| 5 advanced filters   | `CmdLineFilter`, `ModuleFilter`, `PortFilter`, `WindowFilter`, `ParentFilter` |

## License

MIT
