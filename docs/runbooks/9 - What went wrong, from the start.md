## What went wrong, from the start

**The core bug:** Your K3s cluster uses NFS-backed storage (required by the assignment), and NFS is inherently slower than local disk for many small file operations. Two separate components needed time to complete slow first-time setup work, but Kubernetes' default health-check probes (`livenessProbe`/`readinessProbe`) were configured with very short timeouts — `periodSeconds: 30`, `failureThreshold: 3`, no `initialDelaySeconds`. That gave each container roughly 90 seconds to prove it was healthy before Kubernetes killed and restarted it.

**Two things needed way more than 90 seconds on NFS:**
1. **PostgreSQL's first-time `initdb`** — creating the `oc_nextcloud` role, database, and permissions. Every time the probe killed Postgres mid-creation, it left a *half-initialized* data directory. On restart, Postgres saw files already there and skipped setup entirely (`Skipping initialization`) — so the role never actually got created, forever, on every attempt. That's why you kept seeing `role "oc_nextcloud" does not exist`.
2. **Nextcloud's `rsync` file copy** — copying ~800MB / tens of thousands of small PHP/app files from the container image onto the NFS-backed PVC. This alone took 40–50 minutes at the actual measured NFS write speed. The same liveness probe kept killing this mid-copy too.

**A secondary complication:** your NFS export used `async` mode, which acknowledges writes before they're actually durable on disk. Combined with repeated forced kills, this caused genuine data corruption at one point (`pg_filenode.map: No such file`), forcing a full clean reset (delete all PVCs, reinstall from scratch).

**The fix, in order:**
1. Changed `/etc/exports` from `async` to `sync` (write durability, prevents corruption).
2. Increased/removed the database's liveness and readiness probes (`initialDelaySeconds`, `failureThreshold` patched way up) so Postgres could finish `initdb` uninterrupted.
3. Fully **removed** the liveness probe on the `nextcloud-aio-nextcloud` deployment entirely (per your assignment guide's own instructions) — readiness stayed, but nothing could kill the container mid-rsync anymore.
4. Did one final clean `helm uninstall` → delete all PVCs → reinstall, so every component started from a truly fresh state with the fixes already in place.
5. Fixed the Tailscale Funnel target to point at `http://127.0.0.1:11000` (exactly per your assignment's instructions) — earlier experimentation with the LoadBalancer's external IP was a detour that wasn't needed once the actual root cause was resolved.

---

## Sysadmin status-check commands (post-setup)

```bash
# Node and pod health
k3s kubectl get nodes
k3s kubectl get pods -A

# NextCloud-specific pods
k3s kubectl get pods -n default

# Storage — confirm PVCs are bound, not pending
k3s kubectl get pv
k3s kubectl get pvc -n default

# Service/networking layer
k3s kubectl get svc -n default

# Helm release health
helm list -n default
helm status nextcloud-aio -n default

# Live logs if something looks off
k3s kubectl logs -n default deployment/nextcloud-aio-nextcloud --tail=50
k3s kubectl logs -n default deployment/nextcloud-aio-database --tail=50

# NFS export sanity check
cat /etc/exports
sudo exportfs -v

# Tailscale/Funnel status
tailscale status
tailscale funnel status

# Database table check (confirms schema intact)
k3s kubectl exec -n default deployment/nextcloud-aio-database -- psql -U oc_nextcloud -d nextcloud_database -c "\dt" | wc -l
```

---

## What the professor will likely demo (per your assignment PDF)

This is explicit in the doc — **the reboot resilience test**:

> *"Your professor might initiate a reboot of your virtual machine before evaluating your work. You will not be around to make changes to or administer your virtual machine after the reboot. Everything should start working automatically after a reboot."*

So the actual demo is almost certainly:
1. SSH in (or just access via Tailscale — they don't even need SSH necessarily).
2. `sudo reboot` your VM, or trigger a reboot themselves.
3. Wait for it to come back up.
4. Open `https://gilravager.bongo-shiner.ts.net` in a browser — **without touching the terminal at all** — and confirm Nextcloud loads and login works.
5. Log in as the `test` admin account you create, to confirm it has admin rights.
6. Check your personal account for: full name, profile photo, status message, shared file, shared photo, calendar event (this is literally the grading rubric).

**You should test this yourself before the deadline**, exactly as the guide warns:
```bash
sudo reboot
```
Then wait a few minutes, and check:
```bash
k3s kubectl get pods -n default
tailscale funnel status
```
Confirm everything comes back to `Running`/`1/1` on its own, and that the public URL still loads in a browser — **without you re-running any patch or fix commands**. If a reboot breaks it again (e.g., the probe patches we applied via `kubectl patch` are **not persisted in Helm's values** — they'll survive a simple reboot since they're stored in the live deployment spec in etcd, but they'd be lost on another `helm upgrade`/reinstall), that's the one risk area worth double-checking now, while you still have time, rather than discovering it during grading.