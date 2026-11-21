# Ansible Role: docker

Docker Engine from the official repository (`download.docker.com`), with every
package pinned to an exact version and held, and `/etc/docker/daemon.json`
owned whole by the role; optionally the API on TCP with mutual TLS. Debian and
Ubuntu.

## Variables

| Variable | Default | Meaning |
|---|---|---|
| `docker_packages` | `{}` - **required** | Package → exact apt version, each a quoted string. The role refuses an empty map, a missing version and an unquoted one (`aursu.general.validate_package_pins`). |
| `docker_daemon_config` | `{log-driver: local, live-restore: true}` | The whole `daemon.json`. Keys not listed are not kept. |
| `docker_prerequisite_packages` | `[python3-debian]` | Installed first; `deb822_repository` needs it. |
| `docker_repo_url` | `https://download.docker.com/linux/<distribution>` | Repository base. |
| `docker_repo_components` | `[stable]` | |
| `docker_repo_architectures` | `[amd64]` | |
| `docker_tls_hosts` | `[]` | TCP addresses of the API, each a dockerd `-H` value (`tcp://192.168.50.2:2376`). Empty: socket only, and the role removes its listener. |
| `docker_tls_ca_cert` | `""` | PEM of the CA whose client certificates are let in. Required with `docker_tls_hosts`. |
| `docker_tls_server_cert` | `""` | PEM of this host's server certificate. Required with `docker_tls_hosts`. |
| `docker_tls_server_key` | `""` | PEM of its private key; written `0600 root`, never logged. Required with `docker_tls_hosts`. |
| `docker_tls_dir` | `/etc/docker/tls` | Where `ca.pem`, `server.pem`, `server-key.pem` are written. |

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

## TLS listener

```yaml
docker_tls_hosts:
  - tcp://192.168.50.2:2376
docker_tls_ca_cert: "{{ lookup('ansible.builtin.file', 'ca.pem') }}"
docker_tls_server_cert: "{{ lookup('ansible.builtin.file', inventory_hostname ~ '-server.pem') }}"
docker_tls_server_key: "{{ lookup('community.sops.sops', inventory_hostname ~ '-server.key.sops') }}"
```

- **Mutual TLS only.** dockerd lets in any client whose certificate chains to
  `docker_tls_ca_cert` and gives it full control of the host: that CA must
  sign nothing but the clients meant to have it.
- **Hosts and TLS settings are flags of one drop-in**,
  `/etc/systemd/system/docker.service.d/tls-listener.conf`:
  `ExecStart=/usr/bin/dockerd --host=fd:// --host=tcp://… --tlsverify
  --tlscacert=… --tlscert=… --tlskey=… --containerd=…`. dockerd accepts a TCP
  host without TLS, so a listener must never exist without its settings;
  splitting them between flags and `daemon.json` would allow exactly that
  between two tasks. For the same reason `docker_daemon_config` may not hold
  `hosts` or `tls*` keys (dockerd also refuses a setting given both ways).
- **Order:** TLS files → `daemon.json` validated with these flags → live set
  validated (TLS key pair included) → drop-in → restart. A failed validation
  stops the run before the listener appears and before the restart.
- **The way back** is `docker_tls_hosts: []`: the drop-in is removed first,
  then `docker_tls_dir`, then the daemon restarts on its socket only.
- The drop-in repeats the packaged unit's command line (docker-ce 29.x:
  `-H fd:// --containerd=/run/containerd/containerd.sock`); check it when
  moving the pins to a new major version.
- The listener's address must exist when dockerd starts (`network-online.target`).

Not here (yet): users and groups, other daemon flags outside `daemon.json`.
