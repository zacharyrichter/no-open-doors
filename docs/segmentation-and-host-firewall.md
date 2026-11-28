# Segmentation and the host firewall

Two layers decide what dmz can reach and what can reach dmz: the network firewall (OPNsense) in front of everything, and `ufw` on the host itself.

## Why a host firewall here and nowhere else

Every other virtual machine and container in my lab relies on network segmentation alone, and the fleet baseline records that as a deliberate choice (the hypervisors have firewalls of their own). dmz is the exception, because it is the one machine that faces the internet. If the network firewall is ever misconfigured, a second, independent layer on the host is the difference between a mistake and an exposure.

## Outbound: what dmz may reach

The firewall's DMZ interface rules, top to bottom:

| Destination | Port | Purpose |
|---|---|---|
| The firewall's DMZ address | 53 TCP and UDP | DNS |
| The firewall | 123 TCP and UDP | Time |
| Notification server | 443 | Alerts |
| Git server (APPS) | 443 | Pulling stack changes |
| App A (APPS) | 8096 | The first public app |
| App B (APPS) | 5055 | The second public app |
| Monitoring manager | 1514 | Host monitoring agent |
| Monitoring manager | 1515 | Agent enrolment. **Disabled** except while enrolling |
| Management host | one TCP port | Metrics agent |
| The firewall itself | any | **Block.** Nothing else on the firewall is reachable |
| Anything that is not a private network | any | Internet only |

Two things about that table:

- **The last rule is "internet only".** It matches everything that isn't a private address, so dmz can reach the internet and cannot reach any private address that wasn't listed above. A compromised dmz can't scan the lab; it can only reach the handful of services in this table.
- **The enrolment rule is switched off.** It exists so a new agent can register, and it is enabled for the length of an enrolment and then disabled again. A rule that is needed for five minutes a year shouldn't be live the rest of the time.

The two APPS rules are exact: one address and one port each. dmz can reach those two backends and nothing else on that network.

## Inbound: who may connect to dmz

Inbound connections to dmz are allowed only from the management network:

- SSH from the bastion and from the management host.
- One TCP port, from the management host only, for the Docker management agent.

Nothing on the user, apps, backup or any other network can start a connection to dmz, and nothing from the internet can, because there is nothing listening for it ([request-path-and-trust.md](request-path-and-trust.md)).

## On the host: `ufw`

Default deny inbound. The only rules are SSH (`22/tcp`) from the bastion and from the management host.

[`configs/host/etc/ufw/after.rules.snippet`](../configs/host/etc/ufw/after.rules.snippet) adds one more thing, and it exists because of a Docker quirk worth knowing about.

## The one published port

The management agent is the only container that publishes a port on the host. `ufw` can't protect it by itself, and that is easy to miss.

Docker publishes a port by rewriting the destination address (DNAT) and *forwarding* the packet to the container. A forwarded packet never goes through the `INPUT` chain, which is where `ufw`'s rules live. So a `ufw` rule that says "only the management host may reach this port" has no effect on a Docker-published port at all. The port is open to whatever the network allows.

The place to filter forwarded traffic is the `DOCKER-USER` chain, which the `FORWARD` chain consults before Docker's own rules and before `ufw`'s. Two rules go there, in `ufw`'s `after.rules`:

```
-A DOCKER-USER -p tcp -m conntrack --ctorigdstport 8120 --ctdir ORIGINAL -s 10.0.99.6 -j RETURN
-A DOCKER-USER -p tcp -m conntrack --ctorigdstport 8120 --ctdir ORIGINAL -j DROP
```

- `RETURN` for the management host, so it carries on to Docker's normal handling. `DROP` for everyone else.
- `--ctorigdstport 8120` matches the port the client *connected to*, before Docker's DNAT rewrote it. After the rewrite, the destination is the container's own port, which needn't be the same number, so matching the original port is the reliable way to write the rule.
- `--ctdir ORIGINAL` restricts the rule to the client-to-container direction, so the agent's replies aren't dropped by the `DROP` rule.

Because the rules live in `ufw`'s file, `ufw` loads them at boot, before Docker starts. Docker keeps whatever is already in `DOCKER-USER`, so start-up order doesn't matter.

There is no `docker-proxy` process, because the Docker daemon is set to `userland-proxy: false`: published ports are handled only by these `iptables` rules. The host has no global IPv6 address, and Docker publishes the port on IPv4 only, so the port isn't reachable over IPv6 either.

### Four layers on that port

| Layer | What it does |
|---|---|
| Network firewall | Allows the port only from the management host |
| `DOCKER-USER` | Drops everyone but the management host, at the host |
| The agent's own allow-list | Refuses connections from any other address |
| Key pinning | The agent only talks to a management server whose public key it was given |

The agent's remote terminal feature is also turned off: all shell access to the host is SSH, through the bastion.

## Checking it

```bash
# Both DOCKER-USER rules present; the RETURN counter rises while the management host is connected
sudo iptables -L DOCKER-USER -v -n

# ufw: default deny, only the two SSH rules
sudo ufw status numbered
```
