# EasySave

> Backup management software — ProSoft Suite v3.0

EasySave is a .NET 10 application available as both a **WPF GUI** and a **console CLI**. It lets you define, manage, and execute backup jobs in parallel with full/differential strategies, real-time state tracking, pause/resume/stop controls, AES file encryption, and centralized log management.

> **Academic project carried out in a group of 3** as part of the Software Engineering curriculum — CESI 2025/2026

---

## Table of contents

- [Features](#features)
- [Project structure](#project-structure)
- [Requirements](#requirements)
- [Installation](#installation)
- [Usage](#usage)
- [Configuration files](#configuration-files)
- [Docker — log server](#docker--log-server)
- [UML diagrams](#uml-diagrams)
- [Versioning](#versioning)
- [License](#license)

---

## Features

- Unlimited named backup jobs
- Two backup strategies: **full** and **differential**
- **Parallel execution** — all jobs run concurrently
- **Play / Pause / Stop** controls per job, with real-time progress bar
- **Priority file extensions** — priority files are always transferred before others across all running jobs
- **Large file throttle** — at most one file above the configured size threshold transfers at a time
- **AES-256 encryption** via the standalone `CryptoSoft` process (mono-instance enforced by system Mutex)
- **Business software detection** — all jobs pause automatically when the configured process is running, and resume when it stops
- **Centralised logging** — logs can be sent to a Docker log server (`local` / `remote` / `both`)
- Configurable log format: **JSON** (default) or **XML** — switchable at runtime
- Real-time state file (`state.json` or `state.xml`) updated after every file transfer
- Daily log file (`YYYY-MM-DD.json` or `.xml`) written by the `EasyLog` shared library
- Bilingual UI: **French** and **English** (toggle in the GUI header bar or console Settings menu)

---

## Project structure

```
CESI-Genie-Logiciel.slnx
├── EasySave.GUI/           # WPF application — MVVM, views, view models, services
├── EasySave.Console/       # Console application — CLI parsing, interactive menu
├── EasySave.Core/          # Business logic — backup engine, state, config, strategies
├── EasySave.CryptoSoft/    # Standalone AES encryption executable (cryptosoft.exe)
├── EasyLog/                # Shared DLL — JSON/XML logging + Docker forwarding
└── EasySave.Tests/         # xUnit test suite
```

### Key classes

| Class | Project | Role |
|---|---|---|
| `BackupManager` | Core | Orchestrates parallel job execution, pause/resume/stop, business software polling (Singleton) |
| `BackupStrategyBase` | Core | Base class — priority gate, large file semaphore, copy + encrypt |
| `FullBackup` / `DifferentialBackup` | Core | Concrete backup strategies |
| `ConfigManager` | Core | Loads/saves `AppConfig`, migrates v1.0 format (Singleton) |
| `StateManager` | Core | Persists real-time job state to `%AppData%\EasySave\` (Singleton) |
| `CryptoSoftRunner` | Core | Spawns `cryptosoft.exe`, serialises calls via `SemaphoreSlim` |
| `ProcessDetector` | Core | Detects a running business software via `Process.GetProcessesByName()` |
| `AesEncryptor` | CryptoSoft | AES-256 CBC encryption/decryption with PBKDF2 key derivation |
| `Logger` | EasyLog | Appends entries to the daily log file, forwards to Docker (Singleton) |
| `LogForwarder` | EasyLog | POSTs log entries to the remote log server over HTTP |
| `BackupJobsViewModel` | GUI | Drives the jobs list, dispatches play/pause/stop commands |
| `SettingsViewModel` | GUI | Reads and saves all config fields, rewires runtime services on save |
| `LocalizationService` | GUI | Singleton exposing all UI strings in FR/EN, refreshes bindings on language switch |
| `NavigationService` | GUI | Manages the current view via `ContentControl` binding (Singleton) |

---

## Requirements

- Windows 10 / 11
- [.NET 10.0 Runtime](https://dotnet.microsoft.com/download/dotnet/10.0)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/) — optional, for log centralisation
- Visual Studio 2022 / JetBrains Rider (recommended)

---

## Installation

```bash
git clone https://github.com/dhya1404/EasySave.git
cd EasySave
dotnet build
```

---

## Usage

### GUI mode

```bash
dotnet run --project EasySave.GUI
```

- Navigate with the sidebar (**Sauvegardes** / **Paramètres**)
- Use **▶ / ⏸ / ⏹** buttons per job for play/pause/stop
- Toggle language with the **FR / EN** button in the top-right corner

### Console mode

```bash
# Interactive menu
dotnet run --project EasySave.Console

# Run jobs 1 to 3 sequentially
EasySave.Console.exe 1-3

# Run jobs 1 and 3 only
EasySave.Console.exe 1;3
```

---

## Configuration files

All configuration and log files are stored in `%AppData%\EasySave\`.

| File | Path | Description |
|---|---|---|
| `config.json` | `%AppData%\EasySave\config.json` | All application settings |
| `state.json` / `state.xml` | `%AppData%\EasySave\state.*` | Real-time job state |
| `YYYY-MM-DD.json` / `.xml` | `%AppData%\EasySave\logs\` | Daily transfer log |

### `config.json` — v3.0 format

```json
{
  "Jobs": [
    {
      "Name": "Save1",
      "SourceDir": "D:\\project\\source",
      "TargetDir": "E:\\backup\\project",
      "Type": "Full"
    }
  ],
  "LogFormat": "json",
  "LogDestination": "local",
  "LogServerUrl": "http://localhost:5000/logs",
  "BusinessSoftwareName": "calc",
  "LargeFileSizeKb": 1024,
  "PriorityExtensions": [".pdf", ".docx"],
  "EncryptedExtensions": [".txt", ".csv"]
}
```

> **Retrocompat:** a v1.0 config (plain array of jobs) is automatically migrated on first load.

### `YYYY-MM-DD.json` — log entry example

```json
[
  {
    "Name": "Save1",
    "FileSource": "D:\\project\\source\\file.txt",
    "FileTarget": "E:\\backup\\project\\file.txt",
    "FileSize": 174592,
    "FileTransferTime": 38,
    "EncryptionTime": 12,
    "Timestamp": "2026-05-13 14:32:01"
  }
]
```

`FileTransferTime` and `EncryptionTime` are in milliseconds. `EncryptionTime` = 0 (no encryption), > 0 (duration), < 0 (error / mono-instance conflict).

---

## Docker — log server

A Docker log server is bundled for centralised log collection.

```bash
# Start log server + console app
docker compose up

# Start log server only
docker compose up log-server
```

The log server listens on **port 5000** and writes incoming entries to a single daily JSON file, regardless of the number of clients. Set `LogDestination` to `"remote"` or `"both"` and point `LogServerUrl` to `http://<server-ip>:5000/logs`.

---

## UML diagrams

Class and overview diagrams for v3.0 are available in the [`/Diagrams/`](./Diagrams/) directory. Previous versions are archived in [`/Diagrams/Old/`](./Diagrams/Old/).

---

## Versioning

| Version | Date | Description |
|---|---|---|
| 1.0 | **2026-05-15**  | Console app, full/differential backup, EasyLog DLL, JSON logs, up to 5 jobs |
| 1.1 | **2026-04-19**  | Unlimited jobs, JSON/XML log format choice, `EncryptionTime` log field |
| 2.0 | **2026-05-02** | WPF GUI (MVVM), CryptoSoft encryption, business software detection |
| **3.0** | **2026-05-12** | **Parallel execution, Play/Pause/Stop, priority files, large file throttle, AES encryption, FR/EN GUI, Docker log centralisation** |

See [`CHANGELOG.md`](./CHANGELOG.md) for the full history.

---

## License

© ProSoft — All rights reserved.
Unit price: €200 excl. VAT — Annual maintenance contract available (12% of purchase price, SYNTEC index).
