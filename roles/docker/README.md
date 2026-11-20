# Ansible Role: docker

Docker Engine from the official repository (`download.docker.com`), with every
package pinned to an exact version and held, and `/etc/docker/daemon.json`
owned whole by the role. Debian and Ubuntu.

## Variables

| Variable | Default | Meaning |
|---|---|---|
| `docker_packages` | `{}` - **required** | Package → exact apt version. The role refuses an empty map. |
| `docker_daemon_config` | `{log-driver: local, live-restore: true}` | The whole `daemon.json`. Keys not listed are not kept. |
| `docker_prerequisite_packages` | `[python3-debian]` | Installed first; `deb822_repository` needs it. |
| `docker_repo_url` | `https://download.docker.com/linux/<distribution>` | Repository base. |
| `docker_repo_components` | `[stable]` | |
| `docker_repo_architectures` | `[amd64]` | |

```yaml
docker_packages:
  docker-ce: "5:29.9.3-1~ubuntu.24.04~noble"
  docker-ce-cli: "5:29.9.3-1~ubuntu.24.04~noble"
  containerd.io: "2.4.2-4~ubuntu.24.04~noble"
  docker-buildx-plugin: "0.39.1-1~ubuntu.24.04~noble"
  docker-compose-plugin: "5.8.0-1~ubuntu.24.04~noble"
```

## Behaviour

- **The repository key ships with the role** (`files/docker.asc`): fingerprint
  `9DC8 5822 9FC7 DD38 854A E2D8 8D81 803C 0EBF CD88`, "Docker Release (CE deb)",
  sha256 of the file `1500c1f56fa9e26b9b8f42452a553675796ade0807cdce11975eb98170b3a570`
  (checked 2026-11-20). It is embedded in the deb822 source's `Signed-By`, so a run
  downloads nothing to trust and a key change appears in review.
- **Pins are exact and held** (`dpkg_selections: hold`): a plain `apt upgrade`
  cannot move them. A version the repository no longer carries fails the
  install - the intended signal to bump the pin.
- **`daemon.json` is validated before it replaces the live file**
  (`dockerd --validate`), and a change restarts the daemon. With
  `live-restore: true` running containers survive the restart.
- **Check mode on a fresh host stops at the repository task**: `python3-debian`
  is not installed in check mode, and `deb822_repository` cannot run without it.
  The module's `install_python_debian` would install it even in check mode,
  which a dry run must not do.

Not here (yet): the TLS listener for the API, users and groups, daemon flags
outside `daemon.json`.
