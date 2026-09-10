# OpenHands Windows Bridge

**Connect a locally hosted coding agent to your Windows workspace.**

OpenHands Windows Bridge brings together Docker Desktop setup, a Windows runtime
adaptation, and a Python command-line client for working with OpenHands and a local
OpenAI-compatible model server. Start a conversation, follow the agent's actions,
and inspect its changes in a dedicated project folder.

[Quickstart](#quickstart) · [Usage](#usage) · [Configuration](docs/CONFIGURATION.md) ·
[Operations & troubleshooting](docs/OPERATIONS.md) · [Contributing](CONTRIBUTING.md)

> **Project status:** experimental, independent integration targeting the OpenHands
> 1.1 images in this repository. Not an official OpenHands distribution or a
> replacement for the upstream agent platform.

## Overview

Running a containerized coding agent against Windows files involves several moving
parts: host paths, container networking, model configuration, and runtime startup.
This project packages that integration into a setup script and a terminal workflow.

| Component | What it provides |
|---|---|
| Windows setup | Docker checks, workspace path translation, model configuration and app-container creation |
| Runtime adaptation | Conditional Windows/WSL2 port ranges and a workaround for recursive ownership changes on mounted files |
| Python CLI | Conversation creation/resumption, event polling, command output and startup diagnostics |
| Session cleanup | Best-effort conversation shutdown and runtime-container removal when the CLI exits |

Use it to explore local-model coding workflows, experiment with disposable projects,
or study Windows/Docker integration. Model serving, weights, and the coding agent
itself are supplied separately. No performance benchmark or production-readiness
claim is made.

## Quickstart

### Requirements

- Windows with **PowerShell**, **Git**, and **Python 3.10+**.
- **Docker Desktop**, running with Linux containers enabled.
- A running **OpenAI-compatible model server**, its loaded model ID, and enough
  resources for that model. The examples use host port `8000`.
- Network access and disk space for the OpenHands container images.

The main CLI uses only Python's standard library; no `pip install` is required.

### Before starting

Use a disposable workspace without secrets. Setup can modify mounted files and
**removes existing `openhands-app` and matching runtime containers**. Without an
explicit workspace it mounts the repository's **parent directory**; the commands
below override that default. The app has Docker-socket access and publishes port
3000 without a loopback-only bind, so keep it off untrusted networks. Read the
[operational safety notes](docs/OPERATIONS.md#safety-and-data-handling) first.

### 1. Prepare a workspace

Run these commands in PowerShell:

```powershell
git clone https://github.com/Mhrnqaruni/openhands-windows.git
cd openhands-windows

New-Item -ItemType Directory -Force -Path .\workspace | Out-Null
$env:WORKSPACE_DIR = (Resolve-Path .\workspace).Path
```

The local `workspace` folder is ignored by Git and appears as `/workspace` inside
the agent runtime. Set `WORKSPACE_DIR` again whenever you open a new terminal.

### 2. Connect your model

Check Docker and list the models served by your local endpoint:

```powershell
docker info
Invoke-RestMethod http://localhost:8000/v1/models
```

Replace `your-loaded-model-id` with an actual ID from the response:

```powershell
$env:LLM_MODEL = 'your-loaded-model-id'
$env:LLM_BASE_URL = 'http://host.docker.internal:8000/v1'
$env:LLM_API_KEY = 'local-llm'
```

`host.docker.internal` lets the container address the Windows host. `local-llm`
is a placeholder for a server that does not require authentication, not a supplied
credential. See [configuration](docs/CONFIGURATION.md) for other endpoints and keys.

### 3. Launch the app and CLI

```powershell
.\setup.bat
python "open hand/openhands_cli.py"
```

Wait for `OpenHands API is ready` before starting the CLI. The app is available at
`http://localhost:3000`; the CLI prints a conversation ID, waits for its runtime,
and opens a `>` prompt. Enter `exit` to finish.

If setup keeps waiting, press Ctrl+C and follow the
[startup troubleshooting guide](docs/OPERATIONS.md#troubleshooting). The script has
no overall readiness deadline; a successful API check is not an agent acceptance test.

## Usage

### Send a single task

```powershell
python "open hand/openhands_cli.py" --once "Create /workspace/hello.py that prints Hello from OpenHands. Do not install packages or change other files."
```

The CLI displays agent messages and command output. If the task succeeds, the
file should appear at `workspace/hello.py` on Windows. Inspect it before execution:

```powershell
Get-Content .\workspace\hello.py
```

This is an illustrative task, not a recorded benchmark. Agent behavior depends on
the model and runtime; natural-language instructions are not an access-control boundary.

### Keep or resume a session

```powershell
# Leave the runtime available after CLI exit
python "open hand/openhands_cli.py" --no-auto-stop

# Reconnect using the conversation ID printed by the CLI
python "open hand/openhands_cli.py" --conversation YOUR_CONVERSATION_ID --no-auto-stop

# Inspect every available option without starting an agent
python "open hand/openhands_cli.py" --help
```

Normal exit attempts to stop the conversation and remove its runtime, but leaves
the app running. To stop the app without removing its container:

```powershell
docker stop openhands-app
```

See [cleanup and persistence](docs/OPERATIONS.md#cleanup-and-persistence) before
using the broader `cleanup.bat` script. Optional direct-model chat and all CLI
options are covered in [configuration](docs/CONFIGURATION.md).

## Architecture

```text
Windows host
  ├─ setup.bat ── configures Docker Desktop and installs the runtime adaptation
  ├─ Python CLI ── HTTP ── OpenHands app (:3000)
  │                          ├─ local model via host.docker.internal:8000
  │                          └─ Docker socket ── agent runtime container
  └─ dedicated workspace ─────────────────────────── /workspace (read/write)
```

The app coordinates conversations; runtime containers execute agent actions.
The Python CLI is an API client, not the model server or the agent implementation.

| Path | Responsibility |
|---|---|
| [setup.bat](setup.bat) | Setup, model discovery, workspace mapping and app restart |
| [open hand/openhands_cli.py](open%20hand/openhands_cli.py) | Conversations, polling, diagnostics and best-effort cleanup |
| [open hand/docker_runtime.py](open%20hand/docker_runtime.py) | OpenHands-dependent runtime adaptation installed inside the app |
| [cleanup.bat](cleanup.bat) | Force-removal of the app and matching runtime containers |
| [chat.py](chat.py) | Optional chat utility with a fixed local endpoint/model |
| [docs/](docs/) | Configuration, compatibility, operational safety and troubleshooting |

## Compatibility and verification

The setup script selects app image `openhands:1.1` and runtime image
`runtime:1.1-nikolaik` from `docker.openhands.dev/openhands/`. These are image tags,
not digest pins. The runtime adaptation uses upstream internal APIs; changing the
image versions requires compatibility work, not just replacing a tag.

Documentation has been reviewed against the source, local links checked, Python
syntax checked, and CLI help exercised with Python 3.11. There is **no automated
test suite or CI workflow**, and no fresh Docker/model end-to-end result is claimed.
See [known limitations](docs/OPERATIONS.md#known-limitations) for remaining work.

## Contributing

Bug reports should identify the Windows, Docker Desktop, Python, OpenHands image,
and model-server versions involved, with private details removed. Proposed changes
should use a topic branch and explain how they were checked. See
[CONTRIBUTING.md](CONTRIBUTING.md) for lightweight verification and review guidance.

## License and attribution

No project-wide license has been established in a standalone LICENSE file.
The OpenHands-derived runtime's provenance and applicable notices also need
clarification before a formal licensed release; this README grants no new permissions.

[OpenHands](https://github.com/All-Hands-AI/OpenHands) supplies the underlying agent
and runtime. This repository focuses on the Windows setup and CLI integration.
