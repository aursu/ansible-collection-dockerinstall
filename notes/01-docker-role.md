# 01 — Role `docker`: pinned Docker Engine

| | |
|---|---|
| Release | 1.0.0 (2026-11-20) |
| PR | #1 `feature/docker-role` |
| Commits | `255b8d6` bootstrap on `main`; `e2f88a4` skeleton; `8a8b8f6` role; `55892f4` shared package checks |

## What

The first content of the collection: role `aursu.dockerinstall.docker`. It adds
Docker's apt repository, installs Docker Engine at exactly pinned versions and
holds them, and owns `/etc/docker/daemon.json`. Debian and Ubuntu.

## Why — the goal

Heavy work (RPM builds, compilation, unit tests) moves from the Mac to the home
server `media-gw`, which is changed only by Ansible. Docker Engine is the first
thing that work needs. The Puppet module `aursu/puppet-dockerinstall` (v0.39.1)
does this at work; this collection is its Ansible counterpart, and its decisions
were read before writing this one.

## How it works

`tasks/main.yml` runs, in order:

1. **Refuse an unsupported platform** — `os_family` must be `Debian`.
2. **Refuse malformed package specifications** —
   `aursu.general.validate_package_pins` on `docker_packages` (a non-empty map
   of package to exact version, every version a non-empty string) and
   `validate_package_names` on `docker_prerequisite_packages`. The message names
   the variable and the package.
3. **Install the role's own prerequisites** — `python3-debian`, which
   `deb822_repository` needs.
4. `repository.yml` — a deb822 source `docker`: URI
   `https://download.docker.com/linux/<distribution>`, suite = the release
   codename, `components` and `architectures` from variables, and `Signed-By`
   set to the **content** of `files/docker.asc`. A change notifies
   `Refresh the apt cache`, and `meta: flush_handlers` runs it at once.
5. `packages.yml` — `apt` installs `name=version` for every pin
   (`allow_change_held_packages`, so a changed pin can move a held package);
   `dpkg_selections: hold` on each package; `docker` enabled and started.
6. `daemon.yml` — `daemon.json` written from `docker_daemon_config`
   (`to_nice_json`), validated first with `dockerd --validate --config-file %s`;
   a change notifies `Restart docker`.

## Where it is defined

| File | What |
|---|---|
| `roles/docker/defaults/main.yml` | `docker_packages` (required, empty by default), `docker_prerequisite_packages`, `docker_repo_url`, `docker_repo_components`, `docker_repo_architectures`, `docker_daemon_config` (`log-driver: local`, `live-restore: true`) |
| `roles/docker/tasks/main.yml` | the two refusals, prerequisites, and the order of the three task files |
| `roles/docker/tasks/repository.yml` | the deb822 source and the flushed cache refresh |
| `roles/docker/tasks/packages.yml` | pinned install, hold, service |
| `roles/docker/tasks/daemon.yml` | `daemon.json` with `dockerd --validate` |
| `roles/docker/handlers/main.yml` | `Refresh the apt cache`, `Restart docker` |
| `roles/docker/files/docker.asc` | Docker's repository key — fingerprint `9DC858229FC7DD38854AE2D88D81803C0EBFCD88`, file sha256 `1500c1f5…a570`, checked 2026-11-20 |
| `galaxy.yml` | 1.0.0; depends on `aursu.general >= 1.9.0` |
| `.github/workflows/test.yml`, `release.yml`, `scripts/release_preflight.py` | blocking lint on PRs; release by tag with the pre-flight |

## Rejected alternatives

- **Downloading the key at run time** (the Puppet module curls it). The run would
  trust whatever the URL serves, and a key change would never be reviewed.
- **Unpinned packages, or a major-version check.** Platform rule: every version
  exact; a version the repository dropped is the signal to bump the pin.
- **Roles `install` and `config`.** The blocking lint (production profile)
  requires variables prefixed with the role name — `install_packages` says
  nothing. One role, `docker_*` variables, three task files.
- **`deb822_repository`'s `install_python_debian`.** It installs the package even
  in check mode, so a dry run would change the host.
- **A `when: … is changed` task for the cache refresh.** Fails the lint
  (`no-handler`); a plain handler would run after the install, too late.
- **A Jinja chain for the pin check.** Replaced in review by the shared Python
  filter (see `aursu.general` notes/02).

## The lesson carried to the TLS phase

From `dockerinstall::daemon_proxy` at work: dockerd's `--tlsverify` accepts any
client certificate that chains to its CA — it has no notion of whose it is. The
home design keeps the CA **dedicated to the Docker API** (it signs only the
server's and the Mac's certificates), so "signed by this CA" means "the Mac" and
no CN-checking proxy is needed. If that CA ever signs anything else, the proxy
becomes necessary.

## Verified by

`ansible-lint roles/`: 0 failures, 0 warnings, production profile;
`--syntax-check`; `release_preflight.py --tag v1.0.0`. Dry run against an Ubuntu
24.04 host: the checks pass with valid pins, and an unquoted `0.39` stops the run
with `'docker_packages['docker-buildx-plugin']' must be an exact version string,
got 0.39. Quote it in YAML.` A dry run on a fresh host cannot go past
`deb822_repository` (`python3-debian` is not installed in check mode); the first
real apply is the full test.
