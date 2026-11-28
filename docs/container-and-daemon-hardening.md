# Container and Docker daemon hardening

The goal is that a compromised container finds as little as possible to work with: no write access to its own filesystem, no extra privileges, and no way to gain more.

## Per container

| Container | Read-only root | Capabilities | `no-new-privileges` | Other |
|---|---|---|---|---|
| `cloudflared` | yes | all dropped | yes | Runs as the image's unprivileged user. Config mounted read-only |
| Traefik | yes | all dropped except `NET_BIND_SERVICE`, to bind 443 | yes | Config and the bouncer plugin's source mounted read-only. Writes only to its own state and log folder |
| CrowdSec | yes, with a `tmpfs` on `/tmp` | all dropped | yes | Writes only to its state and data folders. Its configuration files and Traefik's logs are mounted read-only |
| Management agent | no | default | yes | Mounts the Docker socket, `/proc` and its stacks folder |

Every container above also has `restart: unless-stopped`.

The management agent is the honest exception. Anything that holds the Docker socket can start a privileged container and own the host, so further restrictions on the agent would be decoration. Instead it is made hard to *reach*: one source address, one key, and a port filtered at four layers ([segmentation-and-host-firewall.md](segmentation-and-host-firewall.md#the-one-published-port)).

When a hardened container won't start, its log names the path or capability it needed. The fix is to grant exactly that, and to add a row to the table above saying why.

## The Docker daemon

Log rotation (`json-file`, 10 MB, three files) and the address pool that keeps unpinned networks away from pinned ones are the fleet's Docker standard: [docker-standards.md](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/docs/docker-standards.md#daemon). dmz adds four settings of its own, because it is the internet-facing host ([`host_vars/dmz.yml`](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/inventory/host_vars/dmz.yml)):

| Setting | Why |
|---|---|
| `no-new-privileges: true` | The default for every container, including one that forgets to ask for it in its compose file |
| `live-restore: true` | Restarting the Docker daemon (a package upgrade, say) leaves the containers running, so the tunnel stays up. Docker only guarantees this across patch releases; on a bigger upgrade the containers may restart, and `unless-stopped` brings them back |
| `userland-proxy: false` | No `docker-proxy` processes. Published ports are handled only by `iptables`, which means the `DOCKER-USER` rules decide who gets in |
| `icc: false` | Stops containers on the default bridge talking to each other. No stack uses that bridge, so it's a backstop |

## Networks

The fleet's rule is that every Docker network is owned by exactly one compose project, and a network that other stacks join is declared `external` and pinned to a subnet wherever an address is referenced elsewhere ([docker-standards.md](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/docs/docker-standards.md#networks)). On dmz:

| Network | Owned by | Why it's pinned |
|---|---|---|
| The tunnel network | The Traefik stack | Traefik's `trustedIPs` has to match its subnet, and Traefik has a fixed address inside it, so the bouncer plugin keeps a single entry in CrowdSec |
| The management network | The management agent's stack | Created on deploy, with no manual step |

The consequence is a deploy order: **Traefik first**, because it creates the network the others join. A stack that joins an `external` network that doesn't exist yet fails immediately and says so.

## Image tags

Image pins follow the fleet's rule ([docker-standards.md](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/docs/docker-standards.md#containers)), and the pins are updated by pull request ([trust-but-verify](https://github.com/zacharyrichter/trust-but-verify)). The configs in this repo show `<pinned-version>` in their place.
