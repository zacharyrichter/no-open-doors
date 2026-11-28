# Postmortem: the log rotation that kept stopping Traefik

**Status:** Resolved
**Impact:** The reverse proxy was left stopped after a Docker restart and stayed down until it was redeployed. Nothing was lost and nothing was exposed.

---

## Summary

Access-log rotation had been added to every host that runs Traefik in Docker. To make Traefik reopen its log file after a rotation, the rotation rule sent it a harmless signal, `USR1`, using `docker kill`.

`docker kill` records the container as *stopped by hand*, whatever signal it sends. A container with `restart: unless-stopped` that Docker believes was stopped by hand is **not** restarted when the Docker daemon next starts. So the next time the daemon stopped (a reboot, a backup in stop mode, a Docker package upgrade), Traefik didn't come back.

---

## What I saw

- **On the internet-facing host,** a `docker-ce` upgrade restarted the daemon and Traefik did not come back. There was no error in the Docker logs. Redeploying the stack from the management tool brought it up cleanly, and a deliberate Docker restart afterwards confirmed that every container returned on its own.
- **That wasn't an explanation.** "Docker skipped it at start-up" described what happened, not why. I turned on `live-restore` the same day, so a daemon restart wouldn't stop the containers in the first place. It was worth having and it is still on, but it addresses one kind of restart, not the reason a container was being skipped.
- **Later, the same outage turned out to be affecting every host that runs Traefik in Docker.** That pointed away from an unlucky upgrade on one machine and toward something all of those hosts shared: the log-rotation rule.

---

## Cause

In effect, the rotation rule ran this after each rotation:

```
docker kill --signal=USR1 traefik
```

Docker's bookkeeping treats that like `docker stop`: the container is flagged as stopped by a person. The flag is the *reason* `unless-stopped` exists (if you stop something yourself, Docker shouldn't fight you), and nothing in the command says "this was only a signal". The process kept running and the flag stayed set, silently, until the next daemon stop.

That also explains the missing error. Docker didn't fail to start Traefik; it correctly declined to restart something it had been told was stopped on purpose.

---

## Fix

The signal is now sent to the container's process on the host, not through Docker:

```
pid=$(docker inspect -f '{{.State.Pid}}' traefik) && kill -USR1 "$pid"
```

`kill` talks to the process directly, so Docker's record of the container is untouched. The rule is part of the fleet's Traefik standard and is applied to every host from one place ([diff-is-the-audit](https://github.com/zacharyrichter/diff-is-the-audit/blob/main/docs/traefik-standards.md#access-log-rotation)). The comment above the command in the rule says why, so nobody tidies it back into `docker kill` later.

To see that a host has the fixed rule:

```bash
# The rule signals the host PID, and only rotates access.log
grep -n 'kill' /etc/logrotate.d/traefik-access
sudo logrotate -d /etc/logrotate.d/traefik-access 2>&1 | grep considering
```

---

## What I'd take from it

1. **A signal isn't a stop, but Docker can't tell the difference.** `docker kill` and `docker stop` both mean "a person did this". Anything an automated job does to a container should go through the narrowest tool that does the job; here that was `kill` on a PID.
2. **A mitigation is not a root cause.** `live-restore` was a good setting and made the first incident less likely to repeat, and it also made the problem look smaller than it was. The question that mattered was "why does Docker think this was stopped?"
3. **A rule applied everywhere is a risk everywhere.** The fault lived in something every Traefik host shared, so the fix belongs in the standard and not on one host. The Traefik standard is applied and audited from one place, so a rule like this can't quietly differ between hosts.
