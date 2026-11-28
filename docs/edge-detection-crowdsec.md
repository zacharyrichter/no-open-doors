# Detection at the edge: CrowdSec

CrowdSec does two jobs here. It watches Traefik's access log and turns patterns in it into decisions (ban or challenge) about an address. And it inspects every request as it arrives, through its AppSec component, and blocks the ones that look like an attack. A plugin inside Traefik enforces both, before a request is forwarded.

```
                        ┌───────────── Traefik ─────────────┐
visitor ─► cloudflared ─► bouncer plugin ─► rest of the chain ─► app
                        │   │        ▲                        │
                        └───┼────────┼────────────────────────┘
               per request  │        │ ban/challenge list,
               AppSec check │        │ pulled every 60 s
                            ▼        │
                        CrowdSec ◄── Traefik access.log (JSON)
                        (AppSec)     parser ─► scenarios ─► profiles ─► decisions
```

Configuration: [`configs/crowdsec/`](../configs/crowdsec/) and the bouncer section of [`fileConfig.yml`](../configs/traefik/config/fileConfig.yml).

## Where the configuration lives

CrowdSec keeps its working state (hub downloads, credentials, its database) in a folder on the host. Every file I've written or changed is instead kept in the stacks repository and mounted read-only over that folder ([`compose.yaml`](../configs/crowdsec/compose.yaml)). A rebuild gets them back with the repository, and nobody can change them by editing the host copy.

## What it reads

Two sources:

- [`acquis.yaml`](../configs/crowdsec/config/acquis.yaml): Traefik's access log, mounted read-only, with the type `traefik`. The log is JSON on purpose, so each request arrives as named fields instead of text to be picked apart.
- [`acquis.d/appsec.yaml`](../configs/crowdsec/config/acquis.d/appsec.yaml): the AppSec listener, which the bouncer plugin calls for every request. It's reachable only on the private Docker network, and only with the bouncer's key.

Installed collections: `crowdsecurity/traefik`, `base-http-scenarios`, `http-cve`, `whitelist-good-actors`, and the two AppSec rule sets, `appsec-virtual-patching` and `appsec-generic-rules`. They're listed in the stack's environment file and installed when the container starts.

## The parser: whose address is this?

Without help, CrowdSec would take the client address from the connection, and every connection comes from `cloudflared`'s container. Every ban would be a ban on the tunnel.

A small local parser ([`00-cf-connecting-ip.yaml`](../configs/crowdsec/config/parsers/s02-enrich/00-cf-connecting-ip.yaml)) fixes that. For access-log events that carry a non-empty `Cf-Connecting-Ip` field, it sets the event's source address to that value. Every decision afterwards is about the real visitor.

It does **not** read `X-Forwarded-For`. That header can be supplied by the client; `Cf-Connecting-Ip` can't ([why](request-path-and-trust.md#why-cf-connecting-ip-can-be-believed)). The access log is also configured to keep no request header except that one, so there is nothing else for a parser to be tempted by.

## The scenario: login brute force

The stock scenarios cover generic web abuse. Failed logins on these two apps needed a scenario of their own, [`app-auth-bf.yaml`](../configs/crowdsec/config/scenarios/app-auth-bf.yaml):

| Part | Setting |
|---|---|
| Matches | `POST` requests answered `401` on App A's sign-in path, and `401` or `403` on App B's sign-in paths (one per sign-in method it offers) |
| Type | Leaky bucket. It holds four failures and drains one every two minutes. A burst of wrong passwords overflows it; an occasional typo never does |
| Blackhole | Five minutes. After an overflow, the same source can't raise another alert straight away |
| Grouping | IPv6 clients are counted, and banned, **per /64**. IPv4 clients are per address |
| Scope | `Range`, so the ban applies to the whole group |

The /64 grouping exists because an IPv6 client usually controls an entire /64 and can change address inside it for free. Banning one address would do nothing; banning the prefix does.

### Why the proxy's log, not the apps' own logs

Both apps log failed logins, and the CrowdSec hub has ready-made rules for them. They can't be used here. The apps sit behind the proxy, and the proxy strips the forwarding headers ([why](request-path-and-trust.md#middleware)), so every failed login in an app's own log comes from the proxy's address. A ban built on that would be a ban on the proxy.

The proxy's access log has the real visitor and the response code, which is all a brute-force rule needs. It also keeps the detection in one place: nothing has to reach into the apps' hosts, and the apps' logs never have to leave them.

This works together with the proxy's own limits on login paths (five requests per three minutes, at most three in flight, and a small maximum body). The limits slow a guesser down immediately; the scenario then blocks them outright.

## AppSec: inspecting each request

The access-log scenarios react after the fact: they need a pattern of requests before they act. AppSec checks each request as it arrives, before it reaches the app.

[`acquis.d/appsec.yaml`](../configs/crowdsec/config/acquis.d/appsec.yaml) uses CrowdSec's default AppSec configuration, which loads two rule sets:

- **Virtual patching:** rules for specific, known vulnerabilities in web software. They block an exploit attempt even if the app behind the proxy hasn't been patched yet.
- **Generic rules:** common attack patterns, such as requests for `.env` files or `.git` directories.

A matching request is blocked straight away. If the same address trips two different rules within a minute, a scenario turns that into a decision about the address ([profiles](#the-profiles-what-happens-on-a-match)).

I left out the broad OWASP Core Rule Set. On API-heavy apps like these it produces too many false positives to block on.

### How it was rolled out

AppSec ran in **detect-only** mode first: matches were logged but nothing was blocked. That took two switches, not one. [`appsec-detect-only.yaml`](../configs/crowdsec/config/appsec-configs/appsec-detect-only.yaml) loads the same rules with `default_remediation: allow`, so a matching request goes through. But the scenario that turns repeated matches into a ban counts every *matched* request, whether it was allowed or not. So that scenario also had to run in [simulation](../configs/crowdsec/config/simulation.yaml), where its decisions are recorded but never enforced.

Once a few hours of real traffic had produced no false positives, both switches were reversed. The detect-only configuration is kept in the repository, so going back is a one-line change plus putting the two scenarios back in simulation.

### Throughput

Every request now waits for an AppSec check, so the number of AppSec workers matters. The default is one, which queues every request behind the one before it. Pages and content got noticeably slower to start loading. One worker per CPU (`routines` in `appsec.yaml`) brought it back to what it is on the LAN.

## The profiles: what happens on a match

[`profiles.yaml`](../configs/crowdsec/config/profiles.yaml) has three rules, checked in order:

| Alert | Decision |
|---|---|
| From the two AppSec scenarios | **Challenge**, not ban. These are the alerts most likely to catch a real visitor by mistake |
| Any other alert about an address | Ban |
| Any other alert about a range (the IPv6 /64s) | Ban |

All three use the same escalating duration: each new decision about the same address lasts four hours longer than the last (4, 8, 12 hours…). It's based on how many decisions that address already has in CrowdSec's database. A one-off false positive clears itself in four hours, and a source that keeps coming back stays out longer each time.

## The bouncer

The bouncer is a Traefik plugin ([crowdsec-bouncer-traefik-plugin](https://github.com/maxlerebourg/crowdsec-bouncer-traefik-plugin)) and runs inside the proxy. It's the first middleware in every chain.

| Setting | Why |
|---|---|
| **Stream mode** | The plugin pulls the full list of decisions every 60 seconds and checks each request's address against it in memory, so the ban check never waits for a call to CrowdSec |
| **Keeps the last list if CrowdSec is down** | If CrowdSec restarts or fails, the plugin keeps enforcing the decisions it already has, instead of refusing every request. Either way the apps stay up |
| **AppSec fails open** | If the AppSec check can't be done (CrowdSec unreachable, an error, or a request body it can't read, which some clients send), the request goes through instead of getting a `403` |
| **Client address from `Cf-Connecting-Ip`** | Trusted only when the request comes from the private Docker network, the same rule the proxy applies |

### Loaded from the repository, not downloaded

Traefik can download plugins when it starts. Here it doesn't: the plugin's source is copied into the stacks repository and mounted read-only. If the plugin catalogue were unreachable at start-up, Traefik would disable plugins, and every router that uses the bouncer would fail. With a local copy, a restart doesn't depend on anything outside my network.

Updates still arrive on their own. A comment in [`traefik.yml`](../configs/traefik/config/traefik.yml) records the plugin version. Renovate watches the plugin's releases against that line, and when there's a new one its pull request also re-copies the plugin's source at the new release tag. The Renovate server allows that copy step and nothing else: three exact commands, with only a plain version tag accepted.

Plugins only load when Traefik starts, so the Traefik stack is set to stop and recreate its container on every deploy. Merging the update and deploying is then a single step that can't be half-done.

### The address that has to stay put

CrowdSec registers a new `<bouncer>@<IP>` entry whenever the bouncer's key is used from an address it hasn't seen. The plugin runs inside Traefik, so the address is Traefik's. Traefik has a **fixed address** on the Docker network, deliberately a high one that Docker's own allocation (which starts at the bottom of the subnet) will never hand out. There is one entry and no new ones appear. It's the same rule as the proxy's trusted subnet: an address that something else depends on is pinned, not left to chance.

### What a blocked visitor sees

Instead of a bare `403`, the plugin serves a page ([`ban.html`](../configs/traefik/config/ban.html)) that says, in plain language, what happened and what to do: wait, or contact the admin. It shows the details an admin needs to find and lift the block:

- the visitor's address
- whether it was a ban or a blocked request
- the time
- Cloudflare's Ray ID, the identifier Cloudflare gives each request

It names no person and links to nothing.

A **challenge** decision serves [`captcha.html`](../configs/traefik/config/captcha.html) instead: a Cloudflare Turnstile check, which for most people is a single checkmark. Passing it lets the *whole address* through for 24 hours, not just that browser. That matters because native TV and phone apps can't show a web challenge. Someone whose household was flagged can pass the check once in a browser on the same network, and their apps work again. The page says so.

## Checking it

```bash
# CrowdSec is reading the access log
docker exec crowdsec cscli metrics show acquisition

# AppSec is running, and what it has processed and blocked
docker exec crowdsec cscli metrics show appsec

# No scenario is in simulation
docker exec crowdsec cscli simulation status

# The local API answers, and the plugin is the only bouncer
docker exec crowdsec cscli lapi status
docker exec crowdsec cscli bouncers list

# Recent alerts, and current decisions (also the first place to look if you've locked yourself out)
docker exec crowdsec cscli alerts list -l 20
docker exec crowdsec cscli decisions list
```

If an app is unreachable from outside, check whether CrowdSec has banned your own address before looking anywhere else.
