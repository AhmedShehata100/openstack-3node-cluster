# OpenStack 3-Node Converged Cluster — Build Runbook

This document is the correct, distilled procedure to reproduce this exact
cluster (control + compute + network + monitoring converged on 3 nodes,
Kolla-Ansible, OVN networking, TrueNAS as shared storage) on a brand-new
set of 3 servers — virtual or physical. Every step below is codified in
the Ansible project in this same directory (`playbook.yml`,
`group_vars/`, `host_vars/`, `templates/`), so most of the work is
"edit a few variables, then run the playbook" rather than manual
commands. Where a step still has to be done by hand (there are a few),
that is called out explicitly.

Read this top to bottom once before starting. Each step says what it
does and why it exists — several steps exist specifically because of a
real outage this cluster hit during testing; those are called out as
**Lesson learned** so you understand why the step isn't optional.

---

## 0. What you need before starting

- 3 servers (VMs or physical), each with:
  - Ubuntu 24.04 (Noble) already installed, SSH reachable
  - 6 network interfaces (see network design below) — 2 can be enough if
    you collapse mgmt/storage/workload onto fewer links, but this
    runbook assumes the 3-bond design actually used
  - At least ~100GB root disk (see Step 9 on resizing later if you start
    smaller)
- 1 external NFS-capable storage box (TrueNAS or similar) reachable from
  all 3 nodes, with a share prepared (Step 5)
- 1 separate control/deploy host (referred to as `ansible-server`
  throughout) that is NOT one of the 3 cluster nodes. This host runs
  Ansible and `kolla-ansible`, and is also where centralized log alerts
  land. It must have SSH key access to all 3 nodes as a passwordless-sudo
  user.

### Network design

Each of the 3 nodes gets 3 bonded (active-backup) network interfaces:

| Bond | Purpose | Example subnet |
|---|---|---|
| `bond-mgmt` | Management, OpenStack API traffic, SSH, inter-service RPC | `192.168.17.0/24` |
| `bond-storage` | NFS traffic to the storage box only | `192.168.10.0/24` |
| `bond-workload` | Tenant/VM network. Keeps its IP only until deploy time, then the IP is stripped and it becomes an OVS bridge port (`br-ex`) | `192.168.18.0/24` |

A floating VIP (`192.168.17.20` in this build, via Keepalived) sits on
`bond-mgmt` and is what every OpenStack client, and every inter-service
call, actually talks to — not any single node's own IP.

---

## 1. Base OS prep on each node

Not part of this Ansible project — done once per node before Ansible
ever touches it:

1. Install Ubuntu 24.04 Server.
2. Enable SSH, allow key-based login.
3. Create a sudo user (this build uses `ansible`) with a passwordless
   sudoers entry and the deploy host's SSH public key installed.
4. **If any node was cloned from another VM's disk image**, regenerate
   `/etc/machine-id` and SSH host keys before first boot, then reboot
   immediately:
   ```bash
   truncate -s 0 /etc/machine-id
   systemd-machine-id-setup
   ln -sf /etc/machine-id /var/lib/dbus/machine-id
   rm -f /etc/ssh/ssh_host_*
   ssh-keygen -A
   reboot
   ```
   **Why:** systemd-networkd derives a bond's MAC address from the
   interface name *and* `/etc/machine-id` — cloned VMs with an identical
   machine-id produce identical bond MACs, which your switch/hypervisor
   will treat as a collision. This must happen, and the node must
   reboot, before networkd ever brings the bond up for the first time.

From here on, everything is driven from `ansible-server`.

---

## 2. Point the Ansible project at your 3 nodes

Edit these files in this project directory:

- **`multinode`** — the inventory. Update the 3 `ansible_host=` IPs under
  `[control]` to your new nodes' management IPs. Leave the rest of the
  file's group structure alone (see the note in Step 12 about why).
- **`host_vars/node1.yml`, `node2.yml`, `node3.yml`** — one file per
  node, giving its management/storage/workload IPs, gateway, and the 6
  NIC device names as seen by the OS (`ip link` on that node). These
  drive the netplan template in Step 3.
- **`group_vars/all.yml`** — shared variables. At minimum update:
  `truenas_nfs_server`, `truenas_nfs_export` (or your storage box's
  equivalents), and `alerts_collector_ip` (the deploy host's own
  management-network IP — see Step 15 for why it must be set explicitly
  rather than auto-detected).

---

## 3. Configure networking (bonds)

```bash
ansible-playbook -i multinode playbook.yml --tags netplan
```

**What it does:** renders `templates/netplan-config.yaml.j2` per node
using that node's `host_vars`, building the 3 bonds described above, and
runs `netplan apply`. Safe to re-run any time — it is a no-op once the
config matches.

At this point `bond-workload` still keeps its IP (`strip_workload_ip` in
`group_vars/all.yml` defaults to `false`) so you can keep working over
it. It only gets stripped right before the actual Kolla-Ansible deploy
(Step 13).

---

## 4. Base host configuration

```bash
ansible-playbook -i multinode playbook.yml
```

Running the playbook with no `--tags` runs everything through the base
host-prep plays in order. The relevant early ones:

- **`/etc/hosts` deployment** (`templates/hosts.j2`): maps node1/2/3
  hostnames to their mgmt IPs.
  **Lesson learned:** this template deliberately does **not** add the
  Debian-standard `127.0.1.1 <hostname>` line. RabbitMQ/Erlang's `epmd`
  resolves a node's own hostname via `gethostbyname()`, which returns
  *every* matching `/etc/hosts` entry; with both `127.0.1.1` and the real
  IP present, Erlang can pick `127.0.1.1` first and then fail to reach
  epmd (which only binds the real IP), causing a RabbitMQ cluster to
  crash-loop. Kolla's own `bootstrap-servers` strips this line for the
  same reason — this template just avoids reintroducing it in the first
  place.
- **`nfs-common` package install** — needed for the NFS mounts later.
- **Swap conversion** — converts an LVM swap logical volume into a 4GB
  swapfile, freeing that LV's space into the root filesystem. Only
  touches a node that still has an LVM `swap` LV; a no-op afterwards.

---

## 5. Prepare shared storage (TrueNAS or equivalent)

Done once, by hand, on the storage box itself (outside this project,
since it's a different system with its own admin interface):

1. Create an NFS share/export (this build uses a ZFS dataset
   `CS-NFS` under a pool, exported over NFS).
2. Set the export's **root-squash mapping** to map root to a real
   root-equivalent user (this build used `maproot_user: root,
   maproot_group: wheel`). Without this, containers performing internal
   `chown` operations on the NFS-mounted directories fail.
3. Restrict the NFS export to the **storage** subnet only (e.g.
   `192.168.10.0/24`), not the management subnet.
4. **Use NFSv3.** This build hit a TrueNAS-specific bug where NFSv4's
   pseudo-root mounts successfully but every subsequent file operation
   fails with `Input/output error`. Every mount in this project pins
   `vers=3` explicitly for that reason — don't switch to v4 without first
   confirming your storage box doesn't have the same issue.

Then from `ansible-server`:

```bash
ansible-playbook -i multinode playbook.yml
```

This includes the play that mounts the export temporarily on node1 and
creates the `cinder/`, `glance/` and `nova/` subdirectories inside it (all
three are needed — Cinder, Glance and Nova each get their own
subdirectory of the same export), plus the plays that mount
`glance/` and `nova/` **persistently** (via `/etc/fstab`, `_netdev`
option) on all 3 nodes at `/var/lib/glance-nfs` and `/var/lib/nova-nfs`
respectively. Cinder's NFS share is configured differently — see Step 11.

---

## 6. (Optional) Grow the root disk

If you started with a small disk and later enlarge the underlying
virtual/physical disk, re-run:

```bash
ansible-playbook -i multinode playbook.yml
```

The disk-growth play (`growpart` → `pvresize` → `lvextend` →
`resize2fs`) is included in every run and is a pure no-op when there's
no free space to claim, so there's no harm leaving it in. If you enlarge
a VM's virtual disk live, you may need to trigger a SCSI rescan first so
the guest kernel actually sees the new size:
```bash
for d in /sys/class/scsi_device/*/device/rescan; do echo 1 | sudo tee $d; done
```

---

## 7. Install Kolla-Ansible on the deploy host

Not templated in this project (it's a one-time setup of the control
host itself, not the cluster nodes). On `ansible-server`:

```bash
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate
pip install -U pip
pip install 'ansible-core>=2.19,<2.20'
pip install kolla-ansible==21.3.1.dev1   # or: pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.2
ansible-galaxy collection install ansible.posix community.general
```

Use the `stable/2025.2` release line. Do not use `2024.1` — it is
end-of-maintenance upstream.

---

## 8. Bootstrap the target nodes for Kolla

```bash
kolla-ansible bootstrap-servers -i multinode --ask-vault-pass
```

**What it does:** installs Docker and other base dependencies on all 3
nodes, and — notably — strips any `127.0.1.1` line from `/etc/hosts` if
present (see Step 4's note; this is Kolla's own built-in safeguard for
the same RabbitMQ issue).

---

## 9. The inventory's non-default group settings

The `multinode` file in this project starts from Kolla-Ansible's
standard multinode template with all 3 nodes placed in `[control]`,
`[network]`, `[compute]` and `[monitoring]`, and `[storage]` left
**empty** (storage is the external NFS box, not Kolla-managed). On top
of that default template, three group overrides matter and must not be
reverted:

- **`[masakari:children]`** must explicitly list `masakari-api`,
  `masakari-engine`, `masakari-hostmonitor`, `masakari-instancemonitor`.
  Missing this crashes the `prechecks` step outright on ansible-core
  2.19+ (`groups['masakari']` has no `.masakari` attribute) when
  `enable_masakari: yes` is set.
- **`[cinder-volume:children]`** points at `control`, not `storage`.
  Since this is a converged topology with an empty `[storage]` group,
  pointing it at `storage` means `cinder-volume` is never scheduled
  anywhere.
- **`[hacluster-remote:children]`** is left **empty**. In a converged
  topology every node is already a full Pacemaker cluster member (it's
  in `[control]`, which pulls in `[hacluster:children]`); also listing
  it under `hacluster-remote` (which the default template points at
  `compute`) makes a node try to be both a full member and a
  remote-only member, and `pacemaker_remote` fails to start.

---

## 10. `globals.yml` — the settings that define this cluster

Copy Kolla-Ansible's sample `/etc/kolla/globals.yml` (from the installed
package) as your starting point, then set:

```yaml
workaround_ansible_issue_8743: yes
kolla_base_distro: "ubuntu"
kolla_base_distro_version: "noble"
distro_python_version: "3.12"
openstack_release: "2025.2"

kolla_internal_vip_address: "192.168.17.20"   # pick an unused mgmt-subnet IP

network_interface: "bond-mgmt"
neutron_external_interface: "bond-workload"
neutron_plugin_agent: "ovn"

enable_cinder: "yes"
enable_cinder_backup: "no"
enable_cinder_backend_nfs: "yes"

enable_masakari: "yes"
enable_valkey: "yes"          # required: Masakari's coordination backend

glance_backend_file: "yes"
glance_file_datadir_volume: "/var/lib/glance-nfs"
nova_instance_datadir_volume: "/var/lib/nova-nfs"
```

Notes on a few of these:
- `neutron_plugin_agent: "ovn"` — this build uses OVN, not classic
  ML2/OVS. OVN's native encapsulation is **Geneve**, not VXLAN; don't
  try to force VXLAN with OVN.
- `nova_instance_datadir_volume` pointing at the shared NFS mount (not
  local disk) is what makes live migration and Masakari evacuation of
  ephemeral-disk instances possible — without it, an instance's disk is
  pinned to whichever node created it.

---

## 11. Cinder's NFS backend config

Separately from Glance/Nova's `/etc/fstab` mounts, Cinder manages its
own NFS mount internally via a shares file:

```bash
mkdir -p /etc/kolla/config/cinder
echo "192.168.10.175:/mnt/Primary-Pool-01/CS-NFS/cinder" > /etc/kolla/config/cinder/nfs_shares
```

(Replace the IP/path with your storage box's actual storage-network
address and export path.)

---

## 12. Generate and vault-encrypt the passwords file

```bash
kolla-genpwd
ansible-vault encrypt /etc/kolla/passwords.yml
```

Every `kolla-ansible` command against this cluster from here on needs
`--ask-vault-pass` (or a vault password file) to decrypt this file.
Remember the vault password — it is not recoverable without it.

---

## 13. Strip the workload interface's IP

Before deploying, `bond-workload` needs to give up its IP address so
Neutron/OVN can turn it into `br-ex`:

```bash
ansible-playbook -i multinode playbook.yml --tags netplan -e strip_workload_ip=true
```

---

## 14. Deploy

```bash
kolla-ansible prechecks -i multinode --ask-vault-pass
kolla-ansible deploy -i multinode --ask-vault-pass
kolla-ansible post-deploy -i multinode --ask-vault-pass
```

`post-deploy` generates `/etc/kolla/admin-openrc.sh`,
`public-openrc.sh` and `clouds.yaml` — the credential files you source
to use the `openstack` CLI against this cluster.

If `deploy` fails partway, it is generally safe to fix the reported
issue and re-run `deploy` again — Kolla-Ansible's plays are idempotent.

---

## 15. Harden against the disk-full/log-runaway failure mode

**Lesson learned (the big one):** `enable_masakari: yes` pulls in
`enable_hacluster: yes`, which deploys Pacemaker + Corosync. Real HA
(fencing, cluster resources) was never configured on top of that on this
build's first pass, so the cluster sat in an infinite DC (Designated
Controller) re-election loop. Nothing capped `pacemaker.log`'s size, and
Fluentd's own log-shipping config doesn't even watch
`hacluster/*.log`, so it silently grew to 16–17GB per node over about 5
days and filled all 3 nodes' root disks to 100%. That cascaded into
MariaDB, ProxySQL, Valkey, Cinder and Fluentd all crashing, and left an
orphaned MariaDB transaction holding a lock on the `services` table for
6 days (killed mid-write by the disk-full I/O error), which kept
Cinder failing on every subsequent restart attempt until the lock was
found and manually cleared.

Run this once, right after the base deploy, on every build:

```bash
ansible-playbook -i multinode playbook.yml
```

This applies (all already included in the main playbook run, no tags
needed):

- A `logrotate` rule capping every file under the `kolla_logs` Docker
  volume at 200MB, checked every 15 minutes (not once a day).
- A faster disk-usage watchdog (`disk-guard.sh`, checked every 5
  minutes) that logs WARNING/CRITICAL as root disk usage climbs past
  80%/90%, and emergency-truncates any single log file that's already
  blown past 300MB as a backstop.
- A **central alert collector**: both of the above log locally via
  `journalctl`/syslog on each node (which nobody watches by default —
  exactly the "nobody noticed" failure mode above, just with a smaller
  blast radius), *and* forward to a single aggregated file,
  `/var/log/kolla-alerts.log`, on the deploy host (`ansible-server`)
  itself — deliberately not one of the 3 cluster nodes, since it must
  stay reachable through the exact kind of event it's watching for. Set
  `alerts_collector_ip` in `group_vars/all.yml` to the deploy host's own
  management-subnet address before running this (see Step 2).
- **`hacluster_pacemaker`/`hacluster_corosync` are left stopped and
  disabled** at the systemd level (`systemctl disable`, not just
  `docker stop` — Kolla generates real systemd units for these
  containers with `RequiredBy=` on each other, so a plain `docker stop`
  gets silently reversed on the next boot or container-manager pass).
  Real instance-level HA (auto-evacuation on node failure) needs proper
  fencing configured first — see Step 18. Until then, this pair staying
  off is a deliberate, safe default, not an oversight.
- **OVS bridge admin-up guard**: a second, unrelated lesson learned the
  hard way — after any full host reboot, `br-ex` (and sometimes
  `br-int`) can come back up in kernel state `DOWN` even though OVS's
  own database shows them correctly configured, silently killing all
  floating-IP/external connectivity until someone notices and runs
  `ip link set br-ex up` by hand. A systemd service + timer now asserts
  both bridges admin-up at boot and every 5 minutes after, permanently.

---

## 16. Smoke test

```bash
source /opt/kolla-venv/bin/activate
source /etc/kolla/admin-openrc.sh

openstack compute service list
openstack volume service list
openstack network agent list
```

All should show every service `up`/alive on all 3 nodes. Then:

1. Upload an image (a cloud image, e.g. Ubuntu 24.04's official
   `.img`, uploaded with `--disk-format qcow2`).
2. Create a network, subnet, router, and floating-IP-capable external
   network.
3. Launch an instance, assign it a floating IP, confirm SSH reachability.
4. Test live migration between two of the 3 nodes:
   ```bash
   openstack server migrate --live-migration --host <other-node> <instance>
   ```

**Known OVN quirk:** if the instance and the network's active OVN
gateway chassis end up on *different* nodes, cross-chassis DNAT'd
floating-IP connections can occasionally get stuck at `SYN_SENT` in
OVN's distributed conntrack table. If a floating IP mysteriously won't
complete a TCP handshake, check which chassis is active
(`ovn-sbctl find Port_Binding type=chassisredirect`) and live-migrate
the instance to co-locate it with that chassis — this reliably resolves
it.

---

## 17. (Optional) Enable additional OpenStack services

The base deploy above only enables the core services (Keystone, Nova,
Neutron, Cinder, Glance, Placement, Masakari, Heat). OpenStack has many
more projects available in the same release, each roughly mapping to a
public-cloud equivalent (Swift↔S3, Octavia↔ELB, Designate↔Route53,
Barbican↔KMS, Magnum↔EKS, Trove↔RDS, Ceilometer+Gnocchi↔CloudWatch,
Watcher↔auto resource optimization, Skyline↔a newer alternative
dashboard to Horizon). None of them require a different OpenStack
release — they're already present in `2025.2`, just disabled by
default.

**Turnkey — just a flag, no extra setup:**
```yaml
enable_prometheus: "yes"
enable_grafana: "yes"
enable_prometheus_openstack_exporter: "yes"
enable_barbican: "yes"
enable_watcher: "yes"
enable_ceilometer: "yes"
enable_gnocchi: "yes"
enable_skyline: "yes"
```

**Needs one extra step right after enabling, before `deploy`:**
```yaml
enable_octavia: "yes"
```
```bash
kolla-ansible octavia-certificates -i multinode --ask-vault-pass
```
Octavia needs TLS certificates for its control-plane/amphora
communication; this generates them. Skipping this step makes `deploy`
fail partway through with a `client.cert-and-key.pem ... Could not find
or access` error.

**Enabled but need further per-service setup before they're actually
usable** (flag alone gets the containers running and registered in
Keystone, but functionality needs more):
```yaml
enable_swift: "yes"      # needs dedicated disks with a specific filesystem
                          # label prepared before deploy — not done in
                          # this build
enable_designate: "yes"  # needs a DNS backend (e.g. bind9) configured
enable_magnum: "yes"     # needs a Fedora CoreOS (or similar) image
                          # uploaded to Glance per COE driver
enable_trove: "yes"      # needs a guest-agent image per supported
                          # datastore (MySQL/MariaDB/PostgreSQL/etc.)
```

After adding flags, re-run:
```bash
kolla-ansible prechecks -i multinode --ask-vault-pass
kolla-ansible deploy -i multinode --ask-vault-pass
```

Also install the matching `openstack` CLI plugins on the deploy host so
you can actually exercise these from the command line:
```bash
source /opt/kolla-venv/bin/activate
pip install python-octaviaclient python-designateclient \
  python-barbicanclient python-magnumclient python-troveclient \
  python-watcherclient python-swiftclient
```

---

## 18. What's deliberately NOT done yet

Real instance-level HA (automatic evacuation of VMs off a node that
actually died) needs Pacemaker/Corosync fencing (STONITH) configured on
top of what this runbook builds. This was attempted once on this build
using SBD watchdog self-fencing and it caused a real, simultaneous
reboot of all 3 nodes — see the long comment block above
`sbd_fencing_mode` in `group_vars/all.yml` for the exact root cause
(host-level `sbd` process couldn't reach the containerized Corosync's
IPC) before attempting this again. `sbd_fencing_mode` is currently
`"disabled"` for that reason. On a future physical-server deployment,
real IPMI/BMC-based fencing (`fence_ipmilan` or a vendor-specific agent)
does not have this problem and is the recommended path — see the
comments in `group_vars/all.yml` for how to switch to it.

---

## 19. Ongoing operations

Run `healthcheck.sh` (same directory as this file) any time you want a
full sweep of container health, core OpenStack service status, disk
usage, and the message-queue/database cluster state across all 3 nodes.
See that script's own header comment for details.
