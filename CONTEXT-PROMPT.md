# OpenStack (Kolla-Ansible) Deployment — Context Handoff

Paste this into a new Claude Desktop conversation to continue this project with full context.

**Role split**: this Claude Desktop conversation is a *second opinion / step narrator*
only — it has no shell access to the actual machines. The real execution happens in a
parallel Claude Code session that has direct SSH access to all 3 nodes, the TrueNAS
box, and the ansible-server. The human relays Claude Code's real command output back
here, and relays this conversation's suggested next steps back to Claude Code to
actually run. So: give concrete next steps and ask for real command output to verify
them, but don't assume anything was done until that output is pasted back — and if a
suggestion here conflicts with a decision already recorded below as final, flag it
as a conflict rather than assuming this conversation's view wins.

## Background

This is a sibling project to `cloudstack-ansible` (a previously built and *live-tested*
CloudStack KVM lab: 3 hosts + a Pacemaker-HA management VM, TrueNAS-backed iSCSI/NFS
storage, monitoring, backups — proven working, including a live failover test).
The goal now is to build the equivalent for **OpenStack**, using the same
"design it, run it against real hardware, fix what breaks, document it" approach —
not a paper design this time, an actually-deployed and tested stack, so it can be
sold to clients.

A prior draft (`/root/openstack-ansible/openstack-ansible`) sketched a 9-node
Kolla-Ansible design (3 controllers + 3 compute + external Ceph) but was never run
against real hardware. **That draft's topology is now superseded** — see "Decisions"
below.

## Real infrastructure (already provisioned and verified working)

Three identical VMs, confirmed consistent as of this session:

| | node1 | node2 | node3 |
|---|---|---|---|
| IP (mgmt) | 192.168.17.11 | 192.168.17.12 | 192.168.17.13 |
| IP (storage) | 192.168.10.11 | 192.168.10.12 | 192.168.10.13 |
| IP (workload) | 192.168.18.11 | 192.168.18.12 | 192.168.18.13 |
| OS | Ubuntu 24.04.5 LTS | same | same |
| Kernel | 6.8.0-139-generic | same | same |
| CPU / RAM | 16 vCPU / 31Gi | same | same |
| Root disk | ~38-39G, LVM `ubuntulinux/root`, ext4 | same | same |
| Swap | 4G swapfile (`/swap.img`) | same | same |
| SSH | user `ansible`, passwordless sudo, key-based | same | same |
| Access from here | `ssh ansible@<ip>` (from this host, 192.168.17.15) works directly | | |

Each node has 3 **bonded** interfaces (active-backup, 2 NICs each):
`bond-mgmt` (192.168.17.x), `bond-storage` (192.168.10.x), `bond-workload` (192.168.18.x).

**Deployment host**: this work happens from a separate VM, hostname `ansible-server`
(192.168.17.15 on the mgmt network, also 192.168.30.15 on another network), Ubuntu
24.04.5, 4 vCPU / 7.7Gi RAM, 38G disk. It already has `ansible`/`ansible-playbook`,
`podman`, `git`, Python 3.12.3 installed, and passwordless SSH (key-based, user
`ansible`) to all 3 nodes already working. This is the `[deployment] localhost` host
in Kolla-Ansible's inventory — kolla-ansible itself installs and runs from here, not
from any of the 3 OpenStack nodes.

**Incidents already found and fixed this session** (worth knowing so they aren't
re-diagnosed from scratch):
- node1's LVM VG/LV were originally named differently (`ubuntu-vg/ubuntu-lv`, 19G unused)
  — renamed to match node2/3 (`ubuntulinux/root`) and grown to use the full disk.
  Hit a real GRUB/`grub-probe` gotcha during the rename (grub-probe reads the *live*
  `/proc/mounts` device string, which doesn't update until reboot — a stale device
  name has to be patched into `grub.cfg` with `sed` post-`update-grub`, not left to
  grub-probe alone, or the box drops to an `(initramfs)` rescue shell on reboot).
- node2 and node3 were cloned without regenerating `/etc/machine-id`, which made
  systemd-networkd generate **identical bond-mgmt MAC addresses** on both (the MAC
  for a bond master with no real hardware is derived from interface name +
  machine-id) — caused intermittent/wrong ARP resolution to node3 from this host.
  Fixed by regenerating machine-id on node3 (`truncate` + `systemd-machine-id-setup`
  + reboot) — now all 3 bonds have distinct MACs.
- Same cloning also left node2 and node3 with **identical SSH host keys** — regenerated
  on node3 (`ssh-keygen -A` after removing `/etc/ssh/ssh_host_*`).
- node2/node3 originally had a separate 1G LVM swap LV instead of a swapfile —
  converted to match node1's swapfile approach (freed the space back into root).

Known remaining cosmetic difference, deliberately left alone: node1's `/boot`
partition is 2G vs 1G on node2/3 (repartitioning risk not worth the benefit).

No docker/podman/ansible installed yet on any node (clean slate for Kolla-Ansible).

There is also a TrueNAS appliance in this environment (`192.168.10.175` on the
storage network — see "Storage: TrueNAS — resolved" below for the full picture,
including an earlier wrong-IP assumption that's now corrected) available for
shared storage.

## Deployment requirements (from the client, as stated)

- **All 3 nodes are identical converged controller+compute nodes** — not a
  separate 3-controller/3-compute split. Every node runs both control-plane
  services and `nova-compute`.
- **HA at the management/control-plane level**: Keystone, Nova/Neutron/Cinder/Glance
  APIs, Horizon, MariaDB/Galera, RabbitMQ, HAProxy+Keepalived VIP — survives losing
  any one of the 3 nodes.
- **HA at the instance level**: if a node dies, VMs running on it are evacuated to a
  surviving node automatically (Masakari, the same role this project's sibling
  `cloudstack-ansible` validated live with SBD fencing — reuse that pattern:
  Corosync/Pacemaker + SBD, no IPMI/BMC available on this hardware either).
- **Per-client network isolation**: each client's network fully isolated from every
  other's (OpenStack multi-tenancy via Neutron, one Keystone Domain per client —
  see the `Multi-tenant production` section already drafted in the old
  `openstack-ansible/README.md` for the intended pattern, still valid).
- **Shared storage via TrueNAS** (not Ceph) — this **differs from the old draft**,
  which assumed a separately-deployed Ceph cluster. Needs a decision: iSCSI+LVM
  (like the CloudStack primary storage) vs NFS, for Cinder/Glance/Nova, backed by
  the existing TrueNAS box.

## References the client gave

- https://github.com/openstack/kolla-ansible
- https://docs.openstack.org/project-deploy-guide/kolla-ansible/2024.1/quickstart.html
  — **stale**: 2024.1 ("Caracal") is now `unmaintained/2024.1` upstream (reached
  End of Maintenance). **Decision: use stable/2025.2 instead** (last fully-released,
  still-maintained branch; 2026.1 exists but is newer/less battle-tested, 2025.1 is
  older). Installed and confirmed working: `kolla-ansible 21.3.1.dev1` in a venv.

## Storage: TrueNAS — resolved

TrueNAS's real IP is **192.168.10.175** (storage network) / 192.168.17.175 (mgmt) —
not `.171`, which was a leftover assumption from the sibling CloudStack project and
turned out to be a stale/unrelated address. Confirmed via the actual vCenter VM
(`TrueNAS-OS`, FreeBSD-based TrueNAS CORE).

**Reusing existing resources** (found already provisioned, essentially empty/unused,
kept their `CS-`-prefixed names rather than renaming):
- iSCSI: zvol `Primary-Pool-01/CS-Lun-01`, 1TB thin-provisioned. iSCSI target
  `cs-lun-01` (id 1), extent `cs-lun-01` (naa
  `0x6589cfc000000e5f9c795f7c4dc0e782`), portal id 1. **Portal was listening on a
  stale/wrong IP `192.168.17.100`** (not even one of TrueNAS's real interfaces) —
  fixed via `midclt call iscsi.portal.update 1 '{"listen": [{"ip":
  "192.168.10.175", "port": 3260}]}'` + `service.restart iscsitarget`. No CHAP
  auth configured (open on the network) — acceptable for now, revisit before
  going fully to production.
- NFS: share id 1, path `/mnt/Primary-Pool-01/CS-NFS`. **Was restricted to
  `192.168.17.0/24`** (mgmt network, wrong) — fixed via `midclt call
  sharing.nfs.update 1 '{"networks": ["192.168.10.0/24"]}'` + `service.reload nfs`.
- Both verified reachable from all 3 nodes over the storage network only (mgmt-network
  access confirmed gone for iSCSI; NFS still answers `showmount` queries on either
  TrueNAS IP because NFS access control is by *client* source IP, not which server
  IP was queried — this is expected, not a leftover hole).

**Root cause of node↔TrueNAS unreachability (now fixed)**: this was never a
firewall/isolation issue. Node1/2/3's `bond-storage` and `bond-workload` vNICs were
plugged into the **wrong vCenter portgroups relative to their names** — bond-storage
(192.168.10.x) was actually wired to the `Farouk-CS-workload` portgroup, and
bond-workload (192.168.18.x) to `Farouk-Horizon-LANSeg-7aeb55f2-...`, while TrueNAS's
storage NIC is on `Farouk-Horizon-LANSeg-...`. Fixed by swapping Network adapter 3&4
↔ 5&6 in vCenter on all 3 nodes to match. Also worth knowing for later network work:
`nfs-common` needed installing on node2/node3 (node1 already had it).

## Other fixes applied this session (beyond the earlier LVM/MAC/SSH-key/swap ones)

- `/etc/hosts` was wrong on all 3: node1 had **stale CloudStack-era entries**
  (`CS-node-001/002/003`, `cs-mgmt.mohsen.org`), node2/node3 had **no entries at
  all** for each other (and didn't even resolve their own hostname, only
  `127.0.1.1 localhost`). Rewritten identically on all 3 with correct
  `192.168.17.11 node1` / `.12 node2` / `.13 node3` entries. Kolla's
  `bootstrap-servers` step will strip the `127.0.1.1 nodeX` line automatically
  (RabbitMQ breaks if hostname resolves to loopback) — just confirm it's gone
  after that step runs, don't need to pre-remove it.
- Pre-flight checks (KVM passthrough, NTP sync, disk/mem, bonds) all passed clean
  on all 3 nodes — see full ad-hoc command output already gathered; no need to
  re-run unless something changes.

## Network design — finalized, do not re-litigate without flagging the conflict

| Kolla setting | Interface | Note |
|---|---|---|
| `network_interface` | `bond-mgmt` (192.168.17.x) | APIs, Galera, RabbitMQ, VIP |
| `tunnel_interface` | `bond-mgmt` (same, doubled up) | Deliberately not a 4th NIC — client declined adding new vNICs |
| Storage (Cinder/Glance ↔ TrueNAS) | `bond-storage` (192.168.10.x) | |
| `neutron_external_interface` | `bond-workload` (192.168.18.x), **repurposed with its IP removed** | Not yet done — netplan still has an IP on it, must be stripped before deploy |
| Neutron backend | **OVN** | Distributed routing, built-in gateway HA — matters once Masakari starts killing nodes |
| Encapsulation | **Geneve** (OVN's native default) | Client initially said "VXLAN" generically; clarified and confirmed Geneve is correct given OVN |
| `kolla_internal_vip_address` | `192.168.17.20` | Confirmed free (no ARP reply) |

**VMware-specific gotcha, client is handling in vCenter**: the portgroup backing
`neutron_external_interface` needs **MAC Learning** (vDS 6.7+) or **Promiscuous
Mode + MAC Address Changes + Forged Transmits = Accept** — otherwise floating
IPs/router MACs (which VMware didn't assign) get silently dropped and floating IPs
"deploy fine" but never pass a packet.

## Current progress / what to do next

Done: venv created (`/opt/kolla-venv`), `kolla-ansible` 2025.2 installed and
verified (`kolla-ansible --version` → `21.3.1.dev1`). In progress: running
`kolla-ansible install-deps`, then populating `/etc/kolla` from
`etc_examples/kolla/*` and copying the `multinode` inventory template out of the
venv's `share/kolla-ansible/ansible/inventory/`.

Next concrete steps, in order:
1. Confirm `install-deps` succeeded.
2. Adapt the copied `multinode` inventory for the **3-node converged** topology
   (all 3 nodes in `[control]`, `[network]`, `[compute]`, `[monitoring]` — not a
   3+3+3 split).
3. Write `/etc/kolla/globals.yml`: `kolla_base_distro_version: noble`,
   `distro_python_version: 3.12`, the interface names and VIP above,
   `neutron_plugin_agent: ovn`, `enable_masakari: yes`, `enable_ceph: no`,
   Cinder/Glance pointed at the TrueNAS iSCSI/NFS resources above (both backends
   are viable now — pick one or both per role, still open).
4. Strip the IP from `bond-workload` in netplan on all 3 nodes before deploying.
5. `kolla-genpwd`, encrypt `/etc/kolla/passwords.yml` with `ansible-vault`.
6. `bootstrap-servers` → confirm `/etc/hosts`'s `127.0.1.1 nodeX` line is gone →
   `prechecks` → `deploy`.
7. Smoke test: image, network, VM, live migration.
8. Masakari + Pacemaker/SBD (reuse the sibling `cloudstack-ansible` SBD pattern),
   then a real node-kill failover test.
9. Multi-tenant Keystone Domains + runbook.
