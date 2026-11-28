# The request path, and why one header can be trusted

## The path

```
visitor → Cloudflare edge → cloudflared (outbound tunnel, dialled from dmz)
        → https://traefik:443 on a private Docker network
        → Traefik router + middleware chain (CrowdSec bouncer plugin first)
        → backend over the firewall's DMZ → APPS rules
             app-a.example.net    → http://10.0.50.80:8096   (App A)
             app-b.example.net    → http://10.0.50.85:5055   (App B)
```

`cloudflared` sends *everything* to `https://traefik:443` and sets `originServerName` to one of the names the wildcard certificate covers. Traefik is configured with strict SNI, so a handshake that carries a name it has no certificate for is refused. Setting the origin name once, in the tunnel's ingress rule, is what makes the tunnel's own connection acceptable ([`cloudflared/config/config.yml`](../configs/cloudflared/config/config.yml)).

## Why nothing but the tunnel can reach the proxy

Traefik has one entry point, `https` on `:443`, and **no port is published on the host**. Port 80 isn't published either. Traefik lives on a Docker network called `warptunnel`, which `cloudflared` and CrowdSec's container also join.

So the proxy has no listener on the LAN and none on the internet. The only way for a visitor's request to reach it is through the tunnel, and the only way into the tunnel is through Cloudflare.

## Why `Cf-Connecting-Ip` can be believed

Cloudflare adds the visitor's real address in `Cf-Connecting-Ip`. In an ordinary setup you can't trust a header like that: a client can send any header it likes. It is safe here for one reason:

> **A client can only reach Traefik through Cloudflare, and Cloudflare sets that header itself.**

Remove that property (publish a port, or put another host on the network) and the header becomes forgeable and every decision built on it is wrong. So the property is enforced, not assumed:

- `forwardedHeaders.trustedIPs` is set to the Docker network's subnet, so Traefik only honours forwarding headers from the tunnel.
- That subnet is **pinned** in the compose file that creates the network. The `trustedIPs` value has to match it, so the two are kept next to each other with a comment in each. An address that is referenced elsewhere is never left to Docker to choose.
- The network is **owned by the Traefik stack**, and every other stack declares it `external`. That is why Traefik is deployed first ([rebuild.md](rebuild.md)).

Everything that needs a client address reads `Cf-Connecting-Ip`: the bouncer, the rate limits, the limit on open login requests, and CrowdSec's decisions ([edge-detection-crowdsec.md](edge-detection-crowdsec.md)). Nothing reads `X-Forwarded-For`.

## Middleware

| Middleware | What it does |
|---|---|
| `crowdsec-bouncer` | The CrowdSec plugin. Rejects addresses with a ban (or sends them to a challenge), and has CrowdSec's AppSec check the request. Runs first in every chain, so a banned address is stopped before anything else is spent on it ([details](edge-detection-crowdsec.md#the-bouncer)) |
| `strip-proxy-headers` | Removes `X-Real-IP`, `X-Forwarded-For` and `X-Forwarded-Host`, and sets `X-Forwarded-Proto: https`. An app never sees a client-supplied address |
| rate limit | Per `Cf-Connecting-Ip`. See below |
| `auth-inflight` | Login paths only: at most three requests open at once per address, so a client can't hold many slow requests open |
| `auth-body-limit` | Login paths only: request bodies over 64 KB get `413`. A login is a few hundred bytes. Runs first on those paths, so an oversized body is refused before anything reads it |
| `securityHeaders` | HSTS (one year, subdomains, preload), `nosniff`, a referrer policy, `SAMEORIGIN` framing, a permissions policy and cross-origin policies. Hides the `Server` header. The old `X-XSS-Protection` header is deliberately not sent: browsers have dropped the filter it controlled, and the filter could itself be abused |
| `compress` | Skips video, audio, images and event streams, which are already compressed or must not be buffered |
| `deny-external` | Answers `403` to everything. Used only by the admin-blocking routers below |

There are two chains:

| Chain | Used for | Middlewares, in order |
|---|---|---|
| `secured-chain` | Normal traffic | bouncer, strip headers, rate limit of 100 requests per second on average (burst 50), security headers, compress |
| `auth-chain` | Login and password-reset paths | body limit, bouncer, strip headers, rate limit of 5 requests per 3 minutes (burst 5), open-request limit, security headers, compress |

Each app has two routers. One matches the whole host and uses `secured-chain`. A second, at a **higher priority**, matches only that app's login and password-reset paths and uses `auth-chain`. A visitor using the app normally and a visitor guessing a password go through the same proxy with very different limits.

## Admin pages aren't public

Both apps are administered only from inside my network. So each has a third router, at a higher priority still, that matches its admin API and sends it to `deny-external`:

- **App A:** plugins, packages and repositories, scheduled tasks, logs, restart and shutdown, backups, devices, the setup wizard, library setup, API keys, creating users and changing their policies, metadata editing, and metrics.
- **App B:** its whole settings API, except the three settings endpoints that ordinary users' pages call.

I chose the paths from the apps' own source code, and included only endpoints the app itself already restricts to administrators. An ordinary user would be refused on those paths anyway, so blocking them at the proxy can't break anything a non-admin does. What it removes is the chance of reaching an admin endpoint from the internet at all, with a stolen admin password or a future bug in the app's permission checks.

The patterns were tested against the paths ordinary clients use (content requests, progress updates, images, sign-in, public server info) before going live: every admin path matched, and none of the ordinary ones did.

## The proxy's own settings

- **TLS:** a Let's Encrypt wildcard certificate, issued by DNS challenge through Cloudflare, so no inbound port is needed for issuing it either. TLS 1.2 minimum, strict SNI, ECDHE with AEAD cipher suites only, X25519 then P-256.
- **Header aliases:** `aliasHeadersStrategy: delete` drops request headers whose names alias another header (`X_Foo` or `X.Foo` for `X-Foo`). Some backends normalise those names to the dashed form, which would let a client slip a header past a rule that only looks for the dashed one.
- **Dashboard:** off.
- **Logs:** `access.log` is JSON, covers statuses 200 to 599, and drops every request header except `Cf-Connecting-Ip`. CrowdSec reads this file. `traefik.log` is rotated by Traefik itself.

The access log is rotated by logrotate. How that signal is sent matters more than it looks: see [the postmortem](postmortem-the-logrotate-that-stopped-traefik.md).

Configuration: [`traefik.yml`](../configs/traefik/config/traefik.yml) and [`fileConfig.yml`](../configs/traefik/config/fileConfig.yml).
