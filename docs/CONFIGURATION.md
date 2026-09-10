# Configuration reference

[Back to README](../README.md)

## Setup environment

Set variables in the PowerShell session used to run `setup.bat`:

```powershell
$env:WORKSPACE_DIR = 'C:\Projects\agent-demo'
$env:LLM_MODEL = 'your-loaded-model-id'
$env:LLM_BASE_URL = 'http://host.docker.internal:8000/v1'
$env:LLM_API_KEY = 'local-llm'
```

Create the workspace directory first. Use a dedicated local folder containing no
credentials or private documents. These example paths and values are placeholders.

| Variable | Consumer | Default and behavior |
|---|---|---|
| `WORKSPACE_DIR` | Setup | Parent of the repository, including siblings; override explicitly with an existing absolute local Windows folder |
| `LLM_MODEL` | Setup | Discovers IDs at `http://localhost:8000/v1/models`, preferring `gpt-oss-120b`; falls back to that name if discovery fails |
| `LLM_BASE_URL` | Setup | `http://host.docker.internal:8000/v1`; does not change the fixed discovery URL |
| `LLM_API_KEY` | Setup | `local-llm`, a placeholder for unauthenticated local servers |
| `OPENHANDS_URL` | CLI | `http://localhost:3000`; does not change the setup script's published port |

The scripts do not load `.env` files. Environment settings do not retroactively
change an existing app container. Rerunning setup recreates the app and removes
matching runtime containers; read [operations](OPERATIONS.md) first.

For an authenticated model server, supply the key privately through the process
environment. Real credentials can be exposed through shell history, container
metadata or logs; do not paste them into committed examples or support messages.
The setup script embeds environment values in batch commands: arbitrary shell
metacharacters and complex credential values are not validated or safely supported.

## Host and container addresses

- `http://localhost:8000/v1` is the host-side default discovery endpoint.
- `http://host.docker.internal:8000/v1` is the model URL passed to the app container.
- `http://localhost:3000` is the CLI's default OpenHands API URL.

If the model uses a different port or host, set both its actual model ID and the
container-reachable URL. Successful access from Windows does not prove access
from Docker. This project does not configure model-server authentication or networking.

## CLI options

Run `python "open hand/openhands_cli.py" --help` for the implemented interface.

| Option | Default | Purpose |
|---|---|---|
| `--base-url` | `OPENHANDS_URL` or `http://localhost:3000` | Select the OpenHands API endpoint |
| `--conversation` | New conversation | Resume an existing conversation ID |
| `--once` | Interactive prompt | Send one message and exit |
| `--timeout` | `300` seconds | Readiness/response polling deadline |
| `--poll-interval` | `1.0` second | Delay between event polls |
| `--no-auto-stop` | Cleanup enabled | Leave the conversation runtime after CLI exit |
| `--no-wait-ready` | Wait for readiness | Skip the startup wait; troubleshooting use only |

Example with a longer response wait:

```powershell
python "open hand/openhands_cli.py" --timeout 600 --poll-interval 2
```

Use positive timeout/interval values; the CLI does not validate their ranges.
Polling timeouts do not undo commands already executed by the agent. Normal-exit
cleanup is best effort, and startup failure or forced termination can leave a runtime.

## Optional direct-model chat

`chat.py` is a separate diagnostic utility, not an OpenHands client. It hardcodes
`http://localhost:8000/v1` and `gpt-oss-120b`, does not read `LLM_*` settings or send
an authentication key, and has no request timeout. Use it only with that compatible
local configuration.

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install requests
.\.venv\Scripts\python.exe chat.py
```

`requests` is needed only for this utility. It is not pinned by this repository.
Enter `clear` to reset the in-memory conversation, `system <message>` to add a
system message, or `exit` to quit. Do not use private content as a diagnostic prompt.
