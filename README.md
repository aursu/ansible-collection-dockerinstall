# Ansible Collection - aursu.dockerinstall

Docker Engine on a host: the package repository, exactly pinned packages and
the daemon configuration - the Ansible counterpart of the Puppet module
`aursu/puppet-dockerinstall`.

| Kind | Name | What |
|---|---|---|
| role | `docker` | Repository (key shipped with the role), pinned and held packages, `daemon.json` owned whole. See `roles/docker/README.md`. |

Debian and Ubuntu for now.
