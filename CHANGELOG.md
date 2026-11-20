# Changelog

All notable changes to `aursu.dockerinstall` are documented here.

This project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [1.0.0] - 2026-11-20

### Added

- **Role `docker`**: Docker Engine from `download.docker.com` on Debian and Ubuntu.
  - The repository is a deb822 source whose `Signed-By` embeds the key shipped in
    `files/docker.asc` (fingerprint `9DC858229FC7DD38854AE2D88D81803C0EBFCD88`), so
    nothing to trust is downloaded at run time.
  - `docker_packages` maps every package to an exact apt version and is required;
    the packages are held, so only the role changes their versions.
  - `/etc/docker/daemon.json` is owned whole (`docker_daemon_config`, default
    `log-driver: local`, `live-restore: true`), validated with
    `dockerd --validate` before it is written, and a change restarts the daemon.
  - Installs `python3-debian` first, which `deb822_repository` needs.
  - Package specifications are checked with `aursu.general.validate_package_pins` and
    `validate_package_names` (hence `aursu.general >= 1.9.0`): an unquoted YAML version or an
    empty one is refused, and the message names the package.
  - The apt cache is refreshed by a handler flushed right after the repository
    appears, before the packages are installed from it.

- CI: blocking `ansible-lint` (production profile); release by tag with the
  pre-flight from `aursu.lvm_setup`.

[1.0.0]: https://github.com/aursu/ansible-collection-dockerinstall/releases/tag/v1.0.0
