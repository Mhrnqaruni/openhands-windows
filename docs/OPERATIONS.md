# Operations and troubleshooting

[Back to README](../README.md)

## Safety and data handling

This configuration is intended for controlled, single-user experimentation.

- **Workspace access:** setup defaults to mounting the repository's parent folder
  read/write. Always override `WORKSPACE_DIR`; the agent can change or delete files
  in the mounted folder. Review and back up anything you place there.
- **Docker access:** the app receives `/var/run/docker.sock`, allowing broad control
  of the daemon and its containers. A restricted workspace alone does not make this
  configuration a secure isolation boundary.
- **Network exposure:** setup publishes `3000:3000` without a loopback-only address.
  The wrapper adds neither authentication nor TLS. Keep it on a trusted machine
  and network; do not expose it through public forwarding or tunnels.
- **Stored data:** `LOG_ALL_EVENTS=true` is enabled and app state uses the
  `openhands_data` named volume. Agent messages, output and credentials may appear
  in state, logs or Docker metadata. Local model hosting does not itself ensure privacy.
- **Container scope:** setup and cleanup force-remove `openhands-app` and containers
  matching `openhands-runtime-`. Other OpenHands sessions on the same daemon can be
  affected. Do not use these scripts concurrently with unrelated sessions.

Secret-pattern scans are useful checks, not a guarantee that a checkout, its history
or future contributions contain no sensitive data. Never share raw logs or state
when reporting problems.

## Startup behavior

`setup.bat` performs these steps:

1. Check Docker, attempt a `winget` installation if needed, and check daemon access.
2. Check that the CLI and runtime adaptation files exist.
3. Discover or select a model ID.
4. Convert the chosen workspace to Docker Desktop's host-mount path format.
5. Remove existing app and matching runtime containers.
6. Start the OpenHands app with model settings, Docker-socket access and persistent state.
7. Copy the runtime adaptation into the app, restart it, and poll its configuration API.

The script does not explicitly check runtime-copy or restart failures. Its readiness
loop has no overall deadline. If startup keeps waiting, interrupt with Ctrl+C and
inspect the app rather than repeatedly recreating containers.

## Troubleshooting

| Symptom | Check | Next step |
|---|---|---|
| Docker command missing or daemon unavailable | `docker info` | Install/start Docker Desktop and select Linux containers |
| Unexpected model ID | Host `/v1/models` response | Set `LLM_MODEL` explicitly; discovery uses port 8000 even with a custom model URL |
| Model works on Windows but not in the app | Container-reachable endpoint | Check the server binding, firewall and `host.docker.internal` address |
| Setup waits indefinitely | `docker ps -a`, then `docker logs openhands-app` privately | Investigate image, API or runtime-patch errors; API availability does not prove agent readiness |
| CLI reports runtime failure | App and relevant runtime-container logs | Record the exact image/model versions and redact private details |
| Port binding fails | The specific reported port and owning service | Resolve that conflict; do not stop unrelated services indiscriminately |
| Workspace is missing or incorrect | Explicit `WORKSPACE_DIR` and Docker file-sharing permissions | Use an existing local drive folder; complex paths/network shares are not validated |
| CLI exits without removing the runtime | Startup failure, forced exit or cleanup warnings | Inspect containers before performing targeted manual cleanup |

## Cleanup and persistence

To stop only the app without removing its container:

```powershell
docker stop openhands-app
```

To resume that same app container:

```powershell
docker start openhands-app
```

For broad cleanup, inspect the candidates first:

```powershell
docker ps -a --filter "name=openhands"
# Only if all affected app/runtime sessions may be discarded:
.\cleanup.bat
```

The cleanup script force-removes the named app and matching runtime containers.
Unpersisted container data can be lost. The `openhands_data` volume and files in
the Windows workspace are not deleted. Removing containers therefore does not
erase conversation history or confidential files stored elsewhere.

The CLI's normal-exit cleanup is narrower: it requests conversation stop and
attempts removal of `openhands-runtime-<conversation-id>`. The runtime adaptation
also contains upstream/configuration-dependent container shutdown behavior; this
is not a general guarantee of isolation from every other session.

## Known limitations

- The app and runtime use mutable `1.1` / `1.1-nikolaik` image tags, not digest pins.
  The copied runtime depends on OpenHands internals and is not validated against
  arbitrary newer releases.
- Windows-specific port ranges apply only when the platform test matches Windows
  or a kernel release ending with `microsoft-standard-WSL2`; other Docker/WSL
  environments can follow the alternative ranges in the source.
- The ownership-change workaround replaces a runtime initialization command in
  the container. Changes in upstream file paths or implementation can break it.
- Batch interpolation, unsupported path characters, missing startup deadlines,
  broad cleanup and network exposure need code-level hardening beyond documentation.
- There is no automated test suite or declared cross-platform compatibility matrix.
  This documentation pass did not run Docker, download images or call a model.
- A standalone project license and upstream-derived code notices still need resolution.

These are recorded limitations, not claims that the implementation is ready for
production accounts, untrusted workloads or a formal supported release.
