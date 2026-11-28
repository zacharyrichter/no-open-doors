# Rebuilding the VM, and checking it afterwards

Everything about the host is either in the fleet's configuration management, in the stacks repository, or reproduced by the steps below. Nothing about it lives only on the VM.

## What a rebuild doesn't bring back

| Lost | Effect |
|---|---|
| Traefik's `acme.json` | A new wildcard certificate is issued automatically at first start |
| CrowdSec's database | Starts empty. Current bans and their history are gone, so escalating ban lengths start again from four hours, and the bouncer's key has to be registered again (step 6) |

The stacks' environment files, the tunnel credentials, CrowdSec's own configuration files and the bouncer plugin's source all come back with the stacks repository, so the tunnel reconnects as the same tunnel and CrowdSec starts with the same rules.

## The order matters in three places

1. **Docker's settings before Docker's first container.** The daemon settings are installed before any container starts, so the first start already uses them.
2. **The management agent by hand, before the management tool can reach the VM.** The management tool deploys everything else, but it can't reach a host whose agent isn't running yet.
3. **Traefik before the other stacks.** It creates the Docker network the others join ([why](container-and-daemon-hardening.md#networks)).

## Steps

1. **Create the VM** on its dedicated node, with Debian and the static address on the DMZ VLAN. Apply the fleet baseline: hardening, SSH certificates and the host certificate (see [air-gap-the-keys](https://github.com/zacharyrichter/air-gap-the-keys)), then the base standard and the CIS baseline from [diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit).
2. **Install Docker** from Docker's apt repository, then run the Docker standard for this host before any container is started. It renders `daemon.json` from the fleet base plus dmz's extras ([`host_vars/dmz.yml`](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/inventory/host_vars/dmz.yml)).
3. **Install `ufw`.** Add the two SSH rules (bastion and management host), put the `DOCKER-USER` rules from [`after.rules.snippet`](../configs/host/etc/ufw/after.rules.snippet) into `/etc/ufw/after.rules`, and enable it. Check that `/etc/ufw/sysctl.conf` has `log_martians=1`: `ufw` applies that file after the system's own sysctl settings, and its stock value is `0`.
4. **Install the monitoring agents.** To enrol the Wazuh agent, enable the disabled enrolment rule in the network firewall, enrol, and disable it again.
5. **Clone the stacks repository** to the management agent's stacks folder and start the agent by hand (`docker compose up -d` in its folder). Then reconnect the VM in the management tool, which manages the stacks from here on.
6. **Deploy the stacks:** Traefik first, then CrowdSec and cloudflared. Register the bouncer plugin's key, which the Traefik stack's environment file already holds (`CROWDSEC_LAPI_KEY`):

   ```bash
   docker exec crowdsec cscli bouncers add traefik-plugin -k <key from the Traefik stack's .env>
   ```

   Until the key is registered, the plugin can't fetch decisions. Requests still go through (the plugin doesn't block everyone when it can't reach CrowdSec), but nothing is banned.

7. **Install the log-rotation rule.** The fleet's Traefik standard installs it ([how it signals Traefik](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/docs/traefik-standards.md#access-log-rotation)).
8. **Run every check below.**

## Verification

```bash
# Containers are up, and hardened as the table in container-and-daemon-hardening.md says
docker ps --format 'table {{.Names}}\t{{.Status}}'
for c in cloudflared traefik crowdsec; do
  docker inspect $c --format '{{.Name}} ro={{.HostConfig.ReadonlyRootfs}} drop={{.HostConfig.CapDrop}} add={{.HostConfig.CapAdd}}'
done

# The tunnel is up (expect several "Registered tunnel connection" lines)
docker logs --tail 30 cloudflared 2>&1 | grep -i registered

# CrowdSec reads the access log, and its local API answers
docker exec crowdsec cscli metrics show acquisition
docker exec crowdsec cscli lapi status
docker exec crowdsec cscli bouncers list    # one entry, traefik-plugin, at Traefik's pinned address

# AppSec is inspecting requests, and nothing is left in simulation
docker exec crowdsec cscli metrics show appsec
docker exec crowdsec cscli simulation status

# The management port is filtered, and the RETURN counter rises while the management host is connected
sudo iptables -L DOCKER-USER -v -n

# Host firewall: only the two SSH rules
sudo ufw status numbered

# Docker daemon settings, and every restart policy is unless-stopped
sudo docker info --format 'live-restore={{.LiveRestoreEnabled}}'
sudo docker inspect --format '{{.Name}} {{.HostConfig.RestartPolicy.Name}}' $(sudo docker ps -aq)

# Log rotation considers access.log only
sudo logrotate -d /etc/logrotate.d/traefik-access 2>&1 | grep considering
```

From outside the network: the public name loads, a new line appears at the end of `access.log`, and an admin path (for example `/System/Logs` on App A) returns `403`.

## If something is wrong

- **A stack change breaks something.** Revert the file in the stacks repository and redeploy.
- **A container won't start after hardening.** Its log names the path or capability it needed. Revert, then record the exception in the hardening table.
- **The `DOCKER-USER` rules break the management tool.** Remove the `:DOCKER-USER` line and the two `-A DOCKER-USER` lines (with their comment) from `/etc/ufw/after.rules`, then `sudo ufw reload`.
- **A container doesn't come back after a Docker restart, and Docker's log has no error.** Docker skips an `unless-stopped` container it has recorded as stopped by hand, which `docker stop`, `docker kill` and any script that calls them all do. Redeploy the stack and the new container starts clean. See [the postmortem](postmortem-the-logrotate-that-stopped-traefik.md).
- **An app is unreachable from outside.** Check, in order: the cloudflared logs (is the tunnel registered?), Traefik's own log, whether CrowdSec has banned your address (`cscli decisions list`), and the firewall rules from the DMZ to the apps network.
