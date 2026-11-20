# 02 — TLS listener for the Docker API

| | |
|---|---|
| Release | 1.1.0 |
| PR | `feature/tls` |

## What

Role `docker` can open the Docker API on TCP with mutual TLS:
`docker_tls_hosts` lists the addresses, `docker_tls_ca_cert`,
`docker_tls_server_cert` and `docker_tls_server_key` carry the PEM contents.
The role writes the files to `/etc/docker/tls/`, adds a systemd drop-in that
runs dockerd with the TCP hosts and the TLS flags, validates everything with
`dockerd --validate`, and restarts. An empty `docker_tls_hosts` removes the
listener.

## Why — the goal

Phase 4 of the home Docker build host: the Docker CLI on the Mac drives the
daemon on `media-gw` over `tcp://192.168.50.2:2376`. The certificates are
issued outside the collection (the consumer's PKI); the collection only
installs what it is given.

## How it works

`tasks/main.yml` order after the packages:

1. **`tls.yml`** — assert: no `hosts` or `tls*` keys in
   `docker_daemon_config`; with `docker_tls_hosts`, all three PEM contents
   present. Then either the TLS files (`ca.pem`, `server.pem` 0644,
   `server-key.pem` 0600 `no_log`) or, with no hosts, removal of the drop-in
   **first** and then of `docker_tls_dir`.
2. **`daemon.yml`** — `daemon.json` (still exactly `docker_daemon_config`)
   with `validate: dockerd --validate --config-file %s <flags>`; then the live
   file is validated with the same flags on every real run, which loads the
   TLS key pair — so a changed certificate alone is checked before the
   restart. Skipped in check mode (nothing new was written).
3. **`tls_listener.yml`** (only with hosts) — the drop-in
   `docker.service.d/tls-listener.conf`: `ExecStart=` reset, then
   `/usr/bin/dockerd` with `docker_dockerd_flags`.
4. Handler `Restart docker` with `daemon_reload: true`.

`vars/main.yml` builds `docker_dockerd_flags` once — `--host=fd://`, one
`--host=` per TCP address, `--tlsverify --tlscacert --tlscert --tlskey` when
there are TCP addresses, `--containerd=…` — and both the drop-in and every
validation use it.

## Where it is defined

| File | What |
|---|---|
| `roles/docker/defaults/main.yml` | `docker_tls_hosts`, the three PEM contents, `docker_tls_dir` |
| `roles/docker/vars/main.yml` | file paths, TLS flags, the full `docker_dockerd_flags` |
| `roles/docker/tasks/tls.yml` | assert; TLS files, or removal |
| `roles/docker/tasks/daemon.yml` | `daemon.json` validated with the flags; live-set validation |
| `roles/docker/tasks/tls_listener.yml` | drop-in directory and drop-in |
| `roles/docker/templates/tls-listener.conf.j2` | the `ExecStart` override |
| `roles/docker/handlers/main.yml` | restart with `daemon_reload` |

## Rejected alternatives

- **TLS settings in `daemon.json`, only `-H` in the drop-in** (the plan's
  first version). Measured on media-gw (dockerd 29.9.3):
  `dockerd --validate` accepts `--host=tcp://…` with an empty `daemon.json`.
  With the two halves in two files there is a window — a failed run, a
  reboot between tasks — in which the daemon would start a TCP listener
  without TLS: root for the whole LAN. In one drop-in they appear and go
  together.
- **`hosts` in `daemon.json`.** The packaged unit passes `-H fd://`; dockerd
  refuses a setting given both as a flag and in the file (measured: "the
  following directives are specified both as a flag and in the configuration
  file: hosts").
- **Reading the packaged `ExecStart` at run time** instead of repeating it.
  Parsing `systemctl show` output is fragile; the packages are pinned, so the
  command line changes only when the pin does, and the README says to check it
  then.
- **Validation in check mode.** The validated files do not exist yet in a
  dry run; validating the old files with the new flags would fail for the
  wrong reason.
- **A separate role for the listener.** `daemon.json` validation must know the
  flags, and the restart must happen once; one role owns both.
- **Listening on all addresses.** The listener belongs to one LAN; the
  address is the consumer's choice, `0.0.0.0` included, but not the default.

## Verified by

- `ansible-lint --profile production`: 0 failures.
- media-gw, dockerd 29.9.3 (read-only): `--validate` rejects an unknown `-H`
  protocol, the `hosts` conflict and a missing key pair; accepts the final
  flag set.
- Dry run against media-gw from `home-manage` with this branch mounted:
  TLS directory and files, `daemon.json` unchanged, drop-in with the expected
  `ExecStart`, restart planned; `changed=6`, nothing failed.
- Apply and the end-to-end checks: in `home-manage` after the release.

## Not covered

- RHEL family (the role is Debian/Ubuntu only).
- Certificate issuance and renewal: the consumer's (`home-manage`
  `playbook_docker_pki.yml`).
