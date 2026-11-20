# Development notes — aursu.dockerinstall

One note per change: what it is, why it was made, how it works, where each part
is defined, and the alternatives that lost. Numbered in the order the changes
landed. Not part of the published collection (`build_ignore`).

| # | Release | Subject |
|---|---|---|
| [01](01-docker-role.md) | 1.0.0 | Role `docker`: repository with the shipped key, pinned and held packages, `daemon.json` |
| [02](02-tls-listener.md) | 1.1.0 | TLS listener: drop-in with hosts and TLS flags together, validation with the flags, removal as the way back |
