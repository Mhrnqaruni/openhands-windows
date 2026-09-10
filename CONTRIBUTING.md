# Contributing

[Back to README](README.md)

Keep changes focused on the Windows setup, runtime integration or CLI. OpenHands
provides the upstream agent; changes to that platform should be distinguished from
changes to this wrapper.

## Before making changes

1. Check `git status` and preserve unrelated work.
2. Create a topic branch from the current `master` branch.
3. Review [operational behavior](docs/OPERATIONS.md), particularly workspace mounts
   and the container-removal scope, before running any setup or cleanup command.
4. Use synthetic data and a disposable workspace. Do not commit `.env` files,
   credentials, virtual environments, logs, browser state or private projects.

## Lightweight checks

From the repository root, these checks do not start Docker or contact a model:

```powershell
python -m py_compile chat.py "open hand/openhands_cli.py" "open hand/docker_runtime.py"
python "open hand/openhands_cli.py" --help
git diff --check
```

Compilation checks syntax only; it does not validate runtime imports or upstream
compatibility. Check local Markdown links and compare documented commands/options
with the scripts. Do not add passing-test or performance badges without evidence.

## Pull requests and reports

Explain the problem, scope, verification performed and any behavior changes.
For runtime-related work, include Windows, Docker Desktop, Python, OpenHands image
and model-server versions. Clearly separate observed results from untested claims.
Use owner review before merging to `master`; do not force-push shared history.

Do not disclose secrets or detailed vulnerabilities in public issues. Ask for a
private reporting channel without including sensitive information. No private
reporting availability or response-time commitment is currently guaranteed.

There is no standalone project LICENSE file. Resolve project licensing and
upstream attribution before proposing a formal licensed release.
