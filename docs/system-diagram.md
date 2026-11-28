# System diagram

## The request path

```
 visitor
   │  HTTPS to app-a.example.net / app-b.example.net
   ▼
 Cloudflare edge  ◄───────────────────────────────┐
                                                   │ outbound tunnel,
 ┌──────────── dmz VM (DMZ VLAN, own node) ───────┼──────────────┐
 │                                                 │             │
 │   cloudflared ──https──► Traefik (443, private Docker network)│
 │                              │                                │
 │                              ├─► bouncer plugin (checks first)│
 │                              │        ▲  │ AppSec check       │
 │                              │  bans  │  ▼ per request        │
 │                              │     CrowdSec ◄─ access.log     │
 │                              ▼                                │
 └──────────────────────────────┼────────────────────────────────┘
                                │ plain HTTP, only to these two
                                │ (firewall rules)
                                ▼
                    APPS VLAN:  App A :8096
                                App B :5055
```

## Who can talk to whom

| From | To | Over | Why |
|---|---|---|---|
| Visitor | Cloudflare | HTTPS | The public names resolve to Cloudflare, not to my network |
| dmz | Cloudflare | Outbound tunnel | The only way a visitor reaches dmz |
| `cloudflared` | Traefik | HTTPS, private Docker network | The only path to the proxy |
| Traefik | App A, App B | HTTP, APPS VLAN | Allowed by two firewall rules, and nothing else on APPS |
| Bastion, management host | dmz | SSH | Administration. The only way in |
| Management host | dmz | One TCP port (Docker agent) | Deploys stack changes. Filtered at four layers |
| dmz | Notification, Git, monitoring hosts | Specific ports | Alerts, pulling stack changes, host monitoring |
| dmz | Internet | Anything | Updates, certificate issuing, the tunnel itself |
| dmz | Every other private network | Blocked | The firewall's last rule is "internet only" |

## The zones

| Zone | What lives there | How it relates to dmz |
|---|---|---|
| DMZ | Only this VM, on its own Proxmox node, in a /29 | Nothing else shares its network, so there is no neighbour to attack sideways |
| APPS | The two backends | dmz can reach exactly two addresses and ports |
| MGMT | Bastion and the management host | The only source of inbound connections to dmz |
| Everything else | Users, backups, printing, the rest | Not reachable from dmz at all |

See [segmentation-and-host-firewall.md](segmentation-and-host-firewall.md) for the rules behind each row.
