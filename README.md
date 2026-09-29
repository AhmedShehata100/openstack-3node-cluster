# OpenStack 3-Node Converged Cluster

Ansible project that deploys a **converged 3-node OpenStack cluster**
(every node runs control + compute + network + monitoring) using
[Kolla-Ansible](https://docs.openstack.org/kolla-ansible/latest/),
OVN networking, and an external TrueNAS box as shared NFS storage for
Glance, Cinder and Nova.

Built and tested against real hardware/VMs, including recovering from a
real production-style outage — several of the plays in this repo exist
specifically because of lessons learned from that (see `RUNBOOK.md`).

## What's in this repo

| Path | Purpose |
|---|---|
| `playbook.yml` | The main Ansible playbook — networking, host prep, TrueNAS NFS mounts, and the operational guards described below |
| `multinode` | Kolla-Ansible inventory for the 3-node converged topology |
| `group_vars/all.yml` | Shared variables (storage endpoints, guard thresholds, fencing mode) |
| `host_vars/node{1,2,3}.yml` | Per-node network config (IPs, NIC names) |
| `templates/` | Jinja2 templates rendered by `playbook.yml` (netplan config, systemd units, logrotate rules, etc.) |
| `RUNBOOK.md` | **Full step-by-step build procedure**, from bare Ubuntu nodes to a working cluster, with an explanation under every step |

Not included here (generated locally, gitignored): `RUNBOOK.pdf` (a
formatted export of `RUNBOOK.md`) and `healthcheck.sh` (a read-only
script that sweeps container/service health across all 3 nodes on
demand).

## Quick start

Read `RUNBOOK.md` top to bottom — it's the authoritative procedure,
written so someone with general Linux/networking/Ansible experience can
follow it and end up with the same cluster. In short:

1. Point `multinode`, `host_vars/`, and `group_vars/all.yml` at your 3
   nodes and storage box.
2. Run `playbook.yml` for base host prep (networking, NFS mounts,
   guards).
3. Install Kolla-Ansible, configure `/etc/kolla/globals.yml`, and run
   `bootstrap-servers` → `prechecks` → `deploy` → `post-deploy`.
4. Re-run `playbook.yml` once more to apply the operational hardening
   (log rotation, disk guard, OVS bridge guard, hacluster left off).

## Design highlights

- **OVN** networking (Geneve encapsulation), not classic ML2/OVS.
- **TrueNAS over NFSv3** as the single shared backend for Glance,
  Cinder *and* Nova instance disks — this is what makes live migration
  and Masakari evacuation work without local-disk pinning.
- **Runaway-log / disk-full guards**: after a real incident where an
  unconfigured Pacemaker cluster's log grew unbounded and filled every
  node's disk, this repo now deploys frequent logrotate, a disk-usage
  watchdog with emergency log truncation, and a central alert collector
  on the deploy host — see the postmortem comment block in `playbook.yml`.
- **OVS bridge admin-up guard**: `br-ex`/`br-int` can come back up in
  kernel state `DOWN` after a host reboot even when OVS's own config is
  correct, silently killing floating-IP connectivity. A systemd
  timer now asserts both bridges up at boot and every 5 minutes after.
- **Instance-level HA (Masakari) fencing is intentionally disabled**
  (`sbd_fencing_mode: "disabled"` in `group_vars/all.yml`). A watchdog
  self-fencing attempt caused a real simultaneous reboot of all 3 nodes
  during testing — see the comment above that variable before
  re-enabling it, and read it again before trying watchdog fencing on
  any new deployment.

## Status

Core services (Keystone, Nova, Neutron, Cinder, Glance, Placement,
Heat, Masakari) plus Prometheus/Grafana, Barbican, Watcher, Ceilometer,
Gnocchi, Skyline, Octavia, Designate, Magnum and Trove are deployed and
verified working. Swift is not yet enabled (needs dedicated,
specifically-labeled disks prepared first). See `RUNBOOK.md` §17-18 for
what's enabled vs. what still needs extra setup.
