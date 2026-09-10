# OpenHands Windows Bridge

Windows setup scripts and a Python CLI for running a self-hosted OpenHands agent
with a local OpenAI-compatible model server through Docker Desktop.

This is an **independent integration project**, not the OpenHands agent itself or
an official Windows distribution. It targets the OpenHands **1.1** container images
specified in `setup.bat`. Compatibility with newer releases is not established.

## What this project provides

- A batch setup script that configures the app container, local-model connection,
  and a Windows workspace mount.
- A runtime adaptation for conditional Windows/WSL2 port ranges and skipping
  recursive ownership changes on Windows-mounted workspaces.
- A standard-library Python CLI for creating or resuming conversations, polling
  agent events, displaying runtime errors, and requesting cleanup on exit.
- An optional direct-model chat utility for the default local GPT-OSS endpoint.

The engineering focus is Windows-to-Docker path handling, host/container
networking, runtime startup diagnostics, and conversation lifecycle management.
There are no measured installation-time or performance comparisons.

## Before you run it

Use a **dedicated, disposable workspace** containing no credentials or private
documents. The agent can execute commands and modify mounted files.

Important behavior in the current scripts:

- Without `WORKSPACE_DIR`, setup mounts the **parent of this repository**, including
  sibling folders, read/write at `/workspace`. The quickstart overrides this.
- Setup forcibly removes `openhands-app` and containers matching
  `openhands-runtime-` before starting. Do not use it alongside unrelated OpenHands
  sessions on the same Docker daemon.
- The app receives the Docker socket, giving it broad control over that daemon.
  Docker is not a complete security boundary for this configuration.
- Port 3000 is published without a loopback-only bind. This wrapper adds no
  authentication or TLS; do not expose it to untrusted networks.
- `LOG_ALL_EVENTS=true` is enabled, and app state persists in `openhands_data`.
  Prompts, outputs and credentials may appear in state, logs or Docker metadata.

## Prerequisites

- Windows with PowerShell, Git, and Docker Desktop configured for Linux containers.
- Docker Desktop installed and running. If missing, setup may attempt installation
  through `winget`; it cannot guarantee installation or startup will complete.
- Python **3.10+** for the documented local workflow. The CLI help was checked with
  Python 3.11; this is not a full platform compatibility test.
- An already-running OpenAI-compatible model server and its loaded model ID.
  This repository does **not** install a model server or download model weights.
- Network access and disk space for the OpenHands container images.

The OpenHands CLI uses Python's standard library. Only the optional `chat.py`
utility needs the third-party `requests` package.

## Quickstart — PowerShell

### 1. Clone and select a safe workspace

```powershell
git clone https://github.com/Mhrnqaruni/openhands-windows.git
cd openhands-windows

New-Item -ItemType Directory -Force -Path .\workspace | Out-Null
$env:WORKSPACE_DIR = (Resolve-Path .\workspace).Path
```

Only place disposable project files in that folder. Always set `WORKSPACE_DIR`
explicitly when opening a new terminal or rerunning setup.

### 2. Check Docker and the model server

```powershell
docker info
Invoke-RestMethod http://localhost:8000/v1/models
```

For the default configuration, the model server listens on the Windows host at
port 8000. OpenHands reaches it from Docker through `host.docker.internal`.
Host access alone does not prove the endpoint is reachable from the container.

Set the **actual model ID** returned by your server, replacing the example below:

```powershell
$env:LLM_MODEL = 'your-loaded-model-id'
$env:LLM_BASE_URL = 'http://host.docker.internal:8000/v1'
$env:LLM_API_KEY = 'local-llm'
```

`local-llm` is a placeholder for a local server that does not require a real key.
If authentication is required, supply the credential privately through the process
environment; never commit it or include it in screenshots or support logs.

### 3. Start OpenHands

```powershell
.\setup.bat
```

Setup starts the app image, copies the runtime adaptation into it, restarts it,
and waits for `/api/options/config`. On success it prints `OpenHands API is ready`.
The app is accessible at `http://localhost:3000`.

The readiness loop currently has **no overall timeout**. If it keeps waiting,
interrupt with Ctrl+C and inspect `docker logs openhands-app` privately. Do not
assume that an API-ready message proves every agent or model operation works.

### 4. Use the CLI

```powershell
# Interactive conversation; enter exit to finish
python "open hand/openhands_cli.py"

# One message, then exit
python "open hand/openhands_cli.py" --once "Create hello.py in /workspace that prints Hello from OpenHands."

# Keep the conversation runtime after the CLI exits
python "open hand/openhands_cli.py" --no-auto-stop
```

Agent output depends on the model and runtime. Inspect any generated files before
running them. By default, normal CLI exit attempts to stop the conversation and
remove its runtime container; it leaves the app container running. Cleanup is not
guaranteed after a startup failure or forced process termination.

## Configuration

Environment settings apply to the process you launch; `.env` files are not loaded
automatically. In PowerShell use `$env:NAME = 'value'`, not Command Prompt's `set`.

| Setting | Used by | Default / behavior |
|---|---|---|
| `WORKSPACE_DIR` | Setup | Parent of the repository; explicitly override with a dedicated absolute Windows folder |
| `LLM_MODEL` | Setup | Tries `http://localhost:8000/v1/models`, preferring `gpt-oss-120b`; falls back to that name if discovery fails |
| `LLM_BASE_URL` | Setup | `http://host.docker.internal:8000/v1`; discovery still uses the fixed host URL above |
| `LLM_API_KEY` | Setup | `local-llm`; a placeholder, not a bundled credential |
| `OPENHANDS_URL` | CLI | `http://localhost:3000`; `--base-url` overrides it, but does not change setup's published port |

Changing environment variables does not reconfigure an existing app container;
rerunning setup recreates it and has the cleanup effects described above. Prefer
simple local drive paths: special characters, UNC paths and arbitrary network
shares have not been validated by the batch path conversion.

Additional CLI options:

```powershell
python "open hand/openhands_cli.py" --help
python "open hand/openhands_cli.py" --conversation YOUR_CONVERSATION_ID --no-auto-stop
python "open hand/openhands_cli.py" --timeout 600 --poll-interval 2
```

`--timeout` controls readiness/response polling, not a guaranteed deadline for
agent-side effects. `--no-wait-ready` bypasses the readiness wait and is intended
for troubleshooting, not the normal startup path.

## Architecture and repository layout

```text
Windows Python CLI → OpenHands app API (:3000) → agent runtime container
                              │                         │
                              └─ local model server     └─ /workspace (read/write)
                                 through Docker's host.docker.internal
```

| File | Responsibility |
|---|---|
| [setup.bat](setup.bat) | Docker checks, model discovery, path conversion, app creation and runtime patch installation |
| [open hand/openhands_cli.py](open%20hand/openhands_cli.py) | Conversation API client, event polling and best-effort runtime cleanup |
| [open hand/docker_runtime.py](open%20hand/docker_runtime.py) | OpenHands-dependent runtime adaptation, executed inside the app container |
| [cleanup.bat](cleanup.bat) | Force-removes the named app and matching runtime containers |
| [chat.py](chat.py) | Optional direct chat with a fixed local endpoint/model |

The configured images are `docker.openhands.dev/openhands/openhands:1.1` and
`docker.openhands.dev/openhands/runtime:1.1-nikolaik`. They are tags, not immutable
digest pins. The copied runtime file depends on that upstream internal API;
changing only an image tag is not a supported upgrade procedure.

The port-range adjustment applies only when `os.name == 'nt'` or the detected
kernel release ends with `microsoft-standard-WSL2`. Docker/WSL environments that
do not match that condition retain the other ranges defined in the file.

### Optional direct-model chat

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install requests
.\.venv\Scripts\python.exe chat.py
```

Unlike setup, `chat.py` hardcodes `http://localhost:8000/v1` and `gpt-oss-120b`;
it does not read `LLM_*` settings or send an authentication key. It also has no
request timeout. Use it only with that compatible local configuration.

## Troubleshooting and cleanup

| Symptom | What to check |
|---|---|
| Docker unavailable | Start Docker Desktop and verify `docker info` before setup |
| Model discovery falls back unexpectedly | Check the fixed host `/v1/models` endpoint; set `LLM_MODEL` explicitly |
| Setup keeps waiting | Interrupt it and inspect app logs; runtime copy/restart failures are not explicitly checked by the script |
| Runtime reports an error | Inspect `docker ps -a` and the relevant container logs; redact prompts, paths and tokens before sharing |
| Port binding fails | Identify the conflicting service; do not indiscriminately stop unrelated services |
| Workspace files are missing | Verify the explicit `WORKSPACE_DIR` and Docker file-sharing access |

To stop only the app, without removing it:

```powershell
docker stop openhands-app
```

For the existing broad cleanup, first inspect the targets:

```powershell
docker ps -a --filter "name=openhands"
# Run only if all matching runtime sessions can be discarded:
.\cleanup.bat
```

Cleanup force-removes `openhands-app` and matching `openhands-runtime-` containers.
Unpersisted container data can be lost. It does not delete the `openhands_data`
named volume or files in the mounted Windows workspace.

## Verification, contributions and licensing

This documentation was checked against the repository scripts. Python syntax and
CLI `--help` can be checked without starting Docker or contacting a model.
There is no automated test suite or CI workflow, and this documentation update
does not claim a fresh end-to-end Docker/model validation or security certification.

For contributions, use a topic branch and describe the Windows, Docker, Python,
OpenHands image and model-server versions involved. Never include credentials,
private workspaces, virtual environments, raw event logs or conversation state.

This repository currently has **no standalone LICENSE file**. The previous README's
MIT badge/link was unsupported and has been removed. A project-wide license and
the provenance/notices for the OpenHands-derived runtime file need to be resolved
before a formal licensed release; this README does not grant new permissions.

OpenHands provides the underlying agent and runtime. This project contributes
the Windows integration and CLI adaptation, not the upstream agent platform.
See the [upstream OpenHands project](https://github.com/All-Hands-AI/OpenHands)
for its documentation and applicable licensing.
