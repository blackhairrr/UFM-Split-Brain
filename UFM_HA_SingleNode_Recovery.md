# UFM HA Single-Node Recovery — Troubleshooting Guide

**Date:** 2026-06-18  
**Cluster:** ufmcluster  
**Surviving node:** hopper-ufm-02 (192.168.12.14)  
**Failed node:** hopper-ufm-01 (192.168.12.32) — intentionally powered off  
**UFM Version:** UFM Enterprise (Docker/HA mode)

---

## Situation Summary

`hopper-ufm-01` was powered off for maintenance. The expectation was that Pacemaker would automatically continue running UFM on `hopper-ufm-02`. Instead, UFM failed to start due to a cascade of dependency and state issues on the surviving node.

---

## Root Causes (in order of discovery)

| # | Root Cause | Symptom |
|---|-----------|---------|
| 1 | STONITH/fencing resources blocking DRBD assignment | DRBD resource stuck, cluster unhealthy |
| 2 | `openibd.service` failed (modules already loaded by kernel) | `ufm-enterprise.service` dependency failure |
| 3 | Unit file typo (`openibd,service` vs `openibd.service`) | systemd invalid argument warning |
| 4 | `/dev/infiniband/umad0` missing inside Docker container | `opensm` unable to start |
| 5 | Stale `failover.flag` file | `ufm-ha-watcher` repeatedly triggering failover |

---

## Step 1 — Remove STONITH Fencing Resources

### Problem
Pacemaker's STONITH (fencing) resources were configured but `fence_node2` was running on the wrong node because `192.168.12.32` was offline. This was blocking normal cluster operation and DRBD resource assignment.

### Diagnosis
```bash
pcs stonith status
pcs constraint location config
```

Output showed:
- Both `fence_node1` and `fence_node2` running on `192.168.12.14`
- `fence_node2` had an `INFINITY` preference for `192.168.12.32` (which was offline)

### Fix
```bash
pcs stonith delete fence_node1
pcs stonith delete fence_node2
pcs resource cleanup
```

> **Note:** Removing fencing disables Pacemaker's split-brain protection. This is acceptable while the peer node is physically powered off, but fencing must be reconfigured before `hopper-ufm-01` comes back online.

### Verify
```bash
pcs status
drbdadm status
```

Expected DRBD result:
```
ha_data role:Primary
  disk:UpToDate
  peer connection:Connecting
```

---

## Step 2 — Fix `openibd.service` Dependency Blocking UFM

### Problem
`ufm-enterprise.service` hard-requires `openibd.service` via `Requires=`. However, `openibd.service` was failing because the InfiniBand kernel modules (`mlx5_ib`, `ib_umad`, etc.) were already loaded and in active use — the `openibd` script could not re-load them. This caused the entire `ufm-enterprise` service to fail with:

```
Job ufm-enterprise.service/start failed with result 'dependency'
```

### Diagnosis
```bash
systemctl status openibd.service
journalctl -u openibd -n 50 --no-pager -o cat
lsmod | grep mlx
ibstat
```

Key finding: `ibstat` showed the HCA was `Active` and `LinkUp` — the IB stack was working fine. The `openibd` failure was a script-level issue, not a hardware issue.

### Fix
Back up and edit the unit file:
```bash
cp /etc/systemd/system/ufm-enterprise.service \
   /etc/systemd/system/ufm-enterprise.service.bak
```

Change in `/etc/systemd/system/ufm-enterprise.service`:
```ini
# BEFORE
Wants=docker.socket
Requires=docker.socket openibd.service

# AFTER
Wants=docker.socket openibd.service
Requires=docker.socket
```

This changes `openibd.service` from a hard requirement to a soft `Wants=` dependency — UFM starts regardless of openibd's state, since IB was already functional.

```bash
systemctl daemon-reload
```

> **Important:** This is a workaround. If UFM is reinstalled or upgraded, this change will be overwritten. Track it in your change log.

---

## Step 3 — Fix Unit File Typo

### Problem
During the edit in Step 2, a comma was accidentally typed instead of a period, resulting in:
```
Wants=docker.socket openibd,service
```

systemd reported:
```
Failed to add dependency on openibd,service, ignoring: Invalid argument
```

### Fix
```bash
sed -i 's/openibd,service/openibd.service/' \
    /etc/systemd/system/ufm-enterprise.service

systemctl daemon-reload
```

### Verify
```bash
head -5 /etc/systemd/system/ufm-enterprise.service
```

Expected:
```
Wants=docker.socket openibd.service
Requires=docker.socket
```

---

## Step 4 — Fix Missing `/dev/infiniband/umad0` Inside Docker Container

### Problem
`opensm` (the Subnet Manager) repeatedly failed to start inside the UFM container. The root cause was that `/dev/infiniband/umad0` (required by opensm to communicate with the HCA's SM agent) was not present inside the running container.

When the container started at `20:12`, the host's `ib_umad` kernel module had not yet fully settled and `/dev/infiniband/umad0` did not exist. Even with `--privileged`, a running Docker container does not dynamically pick up new device nodes created on the host after it starts.

### Diagnosis
```bash
# On host — device present
ls -la /dev/infiniband/
# shows: umad0, issm0, uverbs0

# Inside container — device missing
docker exec ufm ls -la /dev/infiniband/
# shows: only uverbs0
```

Health log confirmation:
```bash
grep -i "opensm\|SMProcessTest" /opt/ufm/files/log/ufmhealth.log
```

Output showed repeated:
```
Process opensm is not running
Running corrective operation RestartSm for test SMProcessTest
```

### Fix
Restart the `ufm-enterprise` service so the container is recreated and picks up the current device state from the host:
```bash
systemctl restart ufm-enterprise
```

### Verify
```bash
# Confirm device now present inside container
docker exec ufm ls -la /dev/infiniband/
# Should now show umad0 and issm0

# Confirm opensm is running
docker exec ufm ps aux | grep opensm
```

---

## Step 5 — Clear Stale `failover.flag`

### Problem
Even after opensm was running correctly, `ufm-ha-watcher` continued to fail immediately on every start with:

```
Failover initiated by ufm
```

This was caused by a stale flag file written during the earlier opensm outage. `ufm_ha_watcher` reads this file on startup and immediately attempts a failover if it exists — regardless of whether the condition that caused it has since been resolved.

### Diagnosis
```bash
find /opt/ufm -iname "*failover*"
```

Found:
```
/opt/ufm/files/conf/failover.flag
```

### Fix
```bash
rm -f /opt/ufm/files/conf/failover.flag
```

Since `/opt/ufm/files/` is on the DRBD-synced volume, this change is reflected inside the container automatically.

### Verify
```bash
systemctl start ufm-ha-watcher
systemctl status ufm-ha-watcher
```

Expected: `Active: active (running)`

---

## Step 6 — Reconcile Pacemaker State

After all individual fixes, clear Pacemaker's stale failure records and allow it to re-evaluate and manage all resources:

```bash
pcs resource cleanup
pcs status
```

### Expected Final State
```
Full List of Resources:
  * Clone Set: ha_data_drbd-clone [ha_data_drbd] (promotable):
    * Promoted: [ hopper-ufm-02 ]
    * Stopped:  [ hopper-ufm-01 ]
  * Resource Group: ufmcluster-grp:
    * ha_data_file_system    Started hopper-ufm-02
    * cluster_virtual_ip     Started hopper-ufm-02
    * ufm-ha-watcher         Started hopper-ufm-02
    * ufm-enterprise         Started hopper-ufm-02
```

### Verify UFM is reachable
```bash
curl -sk https://192.168.12.20/ -o /dev/null -w "%{http_code}\n"
```

Expected: `200` (or `301` redirect to HTTPS login page)

---

## Bringing hopper-ufm-01 Back Online

> **Read this before powering on `192.168.12.32`.**

### What will happen automatically
- Corosync detects the node and marks it `Online`
- DRBD reconnects and begins re-syncing out-of-sync blocks to `.32` (this is automatic and safe — `.14` remains Primary throughout)
- `ha_data_drbd-clone` starts in Secondary role on `.32`
- `ufmcluster-grp` should **remain on `.14`** — the soft location preference (score 50) for `.32` is not strong enough to trigger a resource move on its own

### What you must do before powering on
1. **Reconfigure fencing (STONITH)** — the cluster is currently running without fencing. Re-add fence resources for both nodes before allowing `.32` to rejoin. Refer to the original `ufm_ha_cluster config` installation parameters.
2. **Confirm `failover.flag` is absent on both nodes** — since `/opt/ufm/files/` is DRBD-synced, the deletion in Step 5 will replicate to `.32` automatically once DRBD reconnects.

### Planned failback (if desired)
If you want to return UFM to `.32` as the primary node, do so as a controlled operation:
```bash
ufm_ha_cluster failover
```
Do not rely on automatic relocation — do it deliberately once `.32` is confirmed healthy.

---

## Quick Reference — Key Files

| File | Purpose |
|------|---------|
| `/etc/systemd/system/ufm-enterprise.service` | UFM Docker wrapper — edited to change `Requires=` to `Wants=` for openibd |
| `/etc/systemd/system/ufm-enterprise.service.bak` | Backup of original unit file |
| `/opt/ufm/files/conf/UFMHealthConfiguration.xml` | UFM internal health check config (test thresholds, corrective/give-up actions) |
| `/opt/ufm/files/conf/failover.flag` | Stale failover flag — delete to unblock `ufm-ha-watcher` |
| `/opt/ufm/files/log/ufmhealth.log` | UFM health monitor log — first place to check for `SMProcessTest` or giveup events |
| `/opt/ufm/files/indicators/master_host_name` | Written at container start — indicates which node is currently master |

---

## Lessons Learned

1. When the peer node is intentionally shut down for maintenance, remove or disable STONITH first — or Pacemaker's fencing logic will interfere with resource assignment on the surviving node.
2. `openibd.service` fails gracefully if IB modules are already loaded. The correct fix is to make it a `Wants=` rather than `Requires=` dependency in the UFM unit file, since the IB stack being pre-loaded means the service's objective is already met.
3. Docker containers started before IB devices exist on the host will not pick them up dynamically. If `ufm-enterprise` starts before `ib_umad` is fully loaded, restart the container after the modules settle.
4. UFM's internal health monitor writes a `failover.flag` file when it gives up on a health test. This file must be manually deleted after resolving the underlying condition — the watcher does not self-clear it.
5. Always use `ufm_ha_cluster status` (rather than `pcs status` alone) to get the combined Pacemaker + DRBD + VIP view in one command.
