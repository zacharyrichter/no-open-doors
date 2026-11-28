# no-open-doors: serving apps to the internet without opening a port

One small VM, on its own VLAN and its own Proxmox node, serves two self-hosted web apps (App A and App B in this repo) to people outside my home network. No port is forwarded on my router. Nothing on my LAN, and nothing on the internet, can talk to the reverse proxy directly.

This repo documents *why* it's built the way it is. The guides in [`docs/`](docs/) cover the how, and [`configs/`](configs/) holds the sanitized configuration the host actually runs.

Everything here is written from the live host. Domains, IP addresses and user names are placeholders (`example.net`, `example.com`, `10.0.<vlan>.<host>`). Image tags are shown as `<pinned-version>`: the real files pin an exact version, and [trust-but-verify](https://github.com/zacharyrichter/trust-but-verify) covers how those pins are kept current.

The baseline this host starts from (the Docker daemon settings, the Traefik rules, log rotation) is the fleet's standard and lives in [diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit). This repo covers what is different about a host that faces the internet.

## The constraint

A few apps have to be reachable from the internet. Everything else in my lab must not be. So the question isn't "how do I protect a server that's exposed", it's "how do I expose as little as possible, and make the few things that are exposed hard to abuse".

## Outbound, not inbound

The usual design opens a port on the router and forwards it to a server. This one never does. A small program on the VM (`cloudflared`) dials *out* to Cloudflare and keeps the connection open, and visitors reach the apps through it.

That removes three things at once: the home IP address is never published, the router has no port forwards to misconfigure, and there is no listener on the internet side of my network to scan.

## One reverse proxy, and nothing can reach it but the tunnel

Traefik listens on 443 *inside a private Docker network* and publishes no web port on the host. The only thing that hands it visitors' requests is `cloudflared`. A visitor can't skip the tunnel, and neither can anything on my LAN.

That is what makes the next decision safe.

## One trusted header, and only that one

Cloudflare tells the proxy the visitor's real address in a header (`Cf-Connecting-Ip`). Normally a header like that can be forged by the client. Here it can't, because the only path to Traefik is through Cloudflare's tunnel, and Cloudflare sets it.

So everything that needs to know "who is this?" keys on that one header: the CrowdSec bouncer, the rate limits, and CrowdSec's own decisions. The usual `X-Forwarded-For` family of headers is stripped before a request reaches an app, so no app is ever handed a client-supplied address to believe. Details: [request-path-and-trust.md](docs/request-path-and-trust.md).

## Detect and block at the edge

CrowdSec reads Traefik's access log and turns patterns in it into bans. A bouncer plugin inside Traefik checks every request *before* it is proxied, and CrowdSec's AppSec component inspects each one for known exploits and common attack patterns. A custom scenario catches login brute force on both apps, and groups IPv6 clients by /64, because a single IPv6 user can rotate through addresses inside their prefix.

A blocked visitor gets a page that explains what happened and gives the details an admin needs to lift the block. For the blocks most likely to be a mistake, they get a one-click Cloudflare Turnstile check instead of a ban. Details: [edge-detection-crowdsec.md](docs/edge-detection-crowdsec.md).

## Admin pages stay off the internet

Both apps are administered only from inside my network, so their admin APIs are refused at the proxy for anyone coming through the tunnel. The paths were taken from each app's own source code, and only endpoints the app already restricts to administrators are blocked, so nothing an ordinary user does can be affected. Details: [request-path-and-trust.md](docs/request-path-and-trust.md#admin-pages-arent-public).

## Segmented, with two firewalls

The firewall lets the VM reach only the specific backends and services it needs, plus the internet. Anything that wants to connect *into* the VM has to come from my management network. On the host itself, `ufw` is a second layer, and this is the only virtual machine in my lab that has one, because it is the only one facing the internet. Details: [segmentation-and-host-firewall.md](docs/segmentation-and-host-firewall.md).

## Containers get the least they need

Read-only filesystems, no Linux capabilities, and `no-new-privileges` on every container in the proxy and detection stacks. The one container there that can't be locked down further is the management agent that holds the Docker socket, and the docs say so. Details: [container-and-daemon-hardening.md](docs/container-and-daemon-hardening.md).

## What went wrong

[The log rotation that kept stopping Traefik](docs/postmortem-the-logrotate-that-stopped-traefik.md): a harmless-looking rotation script marked the proxy as "stopped by hand", so Docker refused to bring it back after the next restart. It took a second incident, on a second host, to see that they were the same bug.

## What this doesn't solve

- **Cloudflare sees the traffic.** A tunnel moves trust, it doesn't remove it. Cloudflare terminates TLS for the public names.
- **A compromise of the VM can still reach its two backends.** The firewall limits the VM to exactly those two services on the apps network, and the apps' own logins are what protect them from there.
- **The Docker socket agent is root-equivalent.** Management needs it. It is reachable only from one host, by one key, and the port is filtered at four layers ([segmentation-and-host-firewall.md](docs/segmentation-and-host-firewall.md#the-one-published-port)).
- **Secrets are kept in a private Git repository.** The stacks' environment files and the tunnel credentials are committed to a private repo so a rebuild is quick. That is an accepted trade-off; none of those files are in this repo.
- **Bans start at four hours and grow.** Each new ban for the same source lasts four hours longer than the last. A one-off false positive fixes itself in four hours; a source that keeps coming back stays out longer each time.
- **Native apps can't show a challenge.** A TV or phone app that gets the Turnstile check just fails, the same as a ban. Passing the check once in a browser on the same network lets the whole address back in, and the challenge page says so.

## Layout

| Path | What's in it |
|---|---|
| [`docs/system-diagram.md`](docs/system-diagram.md) | The request path and the network zones |
| [`docs/request-path-and-trust.md`](docs/request-path-and-trust.md) | The tunnel, the proxy, the trusted header, the middleware chains and the admin blocks |
| [`docs/edge-detection-crowdsec.md`](docs/edge-detection-crowdsec.md) | Parser, scenario, AppSec, profiles, the bouncer plugin, and the ban and challenge pages |
| [`docs/segmentation-and-host-firewall.md`](docs/segmentation-and-host-firewall.md) | The OPNsense rules, `ufw`, and the `DOCKER-USER` rule |
| [`docs/container-and-daemon-hardening.md`](docs/container-and-daemon-hardening.md) | Per-container settings, and the Docker daemon settings only dmz needs |
| [`docs/rebuild.md`](docs/rebuild.md) | Rebuilding the VM, and checking it afterwards |
| [`docs/postmortem-the-logrotate-that-stopped-traefik.md`](docs/postmortem-the-logrotate-that-stopped-traefik.md) | The `docker kill` incident |
| [`configs/`](configs/) | Sanitized Traefik, CrowdSec, cloudflared and `ufw` configuration |

## License

MIT. See [LICENSE](LICENSE).
