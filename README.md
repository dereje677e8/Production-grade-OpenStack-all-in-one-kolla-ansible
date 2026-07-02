# INFRA·OPS Cloud Platform

> Single-node private cloud deployment — OpenStack 2024.1 Caracal on Ubuntu 22.04, containerised with Kolla-Ansible 18.4.0, with a custom React operations dashboard backed by a live OpenStack API proxy.

**Region:** Addis-Ababa-AZ1 · **Host:** Ubuntu 22.04 LTS · **Stack:** OpenStack + Docker + Node.js + React + Nginx

---

## Overview

This project demonstrates end-to-end private cloud engineering: automated infrastructure deployment, container orchestration, network virtualisation, block storage provisioning, and a custom operations dashboard consuming live OpenStack REST APIs.

Built and operated as a portfolio infrastructure lab — every component is production-grade, running on real hardware.

---

## Architecture

```
Browser
  │
  ├─── :80  ──────────────────► Apache2 → Horizon (OpenStack native UI)
  │
  └─── :9000 ─────────────────► Nginx
                                   │
                                   ├─ /          → React dashboard (static dist/)
                                   └─ /api/*     → Node.js proxy :4000
                                                      │
                                                      └─ Keystone token auth
                                                         ├─ Nova   (compute)
                                                         ├─ Neutron (network)
                                                         ├─ Cinder  (volumes)
                                                         └─ Glance  (images)

OpenStack Services (containerised via Kolla-Ansible)
  ├─ Keystone    :5000   Identity & token auth
  ├─ Nova        :8774   Compute (QEMU hypervisor)
  ├─ Neutron     :9696   Networking (OVS + br-ex)
  ├─ Glance      :9292   Image registry
  ├─ Cinder      :8776   Block storage (LVM on /dev/sdb)
  ├─ Horizon     :80     Native dashboard
  ├─ MariaDB     :3306   State database
  └─ RabbitMQ    :5672   Message bus

Network topology
  ens6 (172.24.33.58/24) → br-ex (OVS bridge)
    ├─ Provider network: 172.24.33.0/24
    ├─ Floating IP pool: 172.24.33.100–172.24.33.150
    ├─ Tenant network:   192.168.100.0/24 (demo-net)
    └─ dummy0 → neutron external interface

Storage
  /dev/sdb  200G  → Cinder LVM (cinder-volumes VG)
  /dev/sdc  250G  → reserved (Swift / second Cinder pool)
```

---

## Stack

| Layer | Technology |
|---|---|
| OS | Ubuntu 22.04.5 LTS |
| Deployment | Kolla-Ansible 18.4.0 + Ansible Core 2.15 |
| Cloud platform | OpenStack 2024.1 Caracal |
| Containerisation | Docker CE 29.x (30+ containers) |
| Hypervisor | QEMU (KVM-compatible mode) |
| Networking | Open vSwitch (OVS) + Neutron ML2 |
| Block storage | Cinder LVM backend |
| Frontend | React 18 + Vite |
| Backend proxy | Node.js 20 + Express |
| Web server | Nginx 1.18 |
| Config management | Ansible Galaxy collections (openstack.kolla, ansible.posix, community.docker) |

---

## Services running

```bash
$ openstack service list
+------------------+-----------+
| Name             | Type      |
+------------------+-----------+
| keystone         | identity  |
| nova             | compute   |
| neutron          | network   |
| glance           | image     |
| cinderv3         | volumev3  |
| heat             | orchestration |
| placement        | placement |
+------------------+-----------+
```

```bash
$ openstack compute service list
+----+------------------+--------+----------+---------+-------+
| ID | Binary           | Host   | Zone     | Status  | State |
+----+------------------+--------+----------+---------+-------+
|  1 | nova-conductor   | gitlab | internal | enabled | up    |
|  2 | nova-scheduler   | gitlab | internal | enabled | up    |
|  3 | nova-compute     | gitlab | nova     | enabled | up    |
+----+------------------+--------+----------+---------+-------+
```

---

## Custom dashboard

The INFRA·OPS dashboard (`/opt/infraops-dashboard`) is a full-stack custom application built alongside the OpenStack deployment:

**Backend** (`/backend/server.js`)
- Express.js proxy authenticating to Keystone via token-based auth
- Exposes simplified REST endpoints: `/api/instances`, `/api/networks`, `/api/volumes`, `/api/quotas`, `/api/hypervisors`
- Token caching with 55-minute refresh cycle
- Flavor resolution with in-memory cache
- Managed as a systemd service (`infraops-backend.service`)

**Frontend** (`/frontend/src/Dashboard.jsx`)
- React 18 + Vite build
- Live polling every 10 seconds
- Tabs: Compute, Networks, Storage, Topology, Blueprint, Analytics
- Served via Nginx on port 9000
- Branded with INFRA·OPS identity (deep navy + signal teal + alert amber)

```
http://172.24.33.58:9000   →  INFRA·OPS Dashboard
http://172.24.33.58        →  OpenStack Horizon
```

---

## Deployment

### Prerequisites

```bash
# Python virtualenv
python3 -m venv /opt/kolla-venv
source /opt/kolla-venv/bin/activate

# Exact version pair (critical)
pip install 'ansible-core==2.15.12' 'kolla-ansible==18.4.0'

# Ansible Galaxy collections
ansible-galaxy collection install \
  ansible.utils ansible.posix \
  community.general community.docker \
  community.crypto containers.podman \
  --force

# Patch collection branch reference
sed -i 's/stable\/2024.1/unmaintained\/2024.1/' \
  /opt/kolla-venv/share/kolla-ansible/requirements.yml
kolla-ansible install-deps
```

### Storage preparation

```bash
# Cinder LVM backend on /dev/sdb
pvcreate /dev/sdb
vgcreate cinder-volumes /dev/sdb

# Neutron external dummy interface (keeps management NIC safe)
ip link add dummy0 type dummy
ip link set dummy0 up
```

### Key globals.yml settings

```yaml
kolla_base_distro: "ubuntu"
openstack_release: "2024.1"
kolla_internal_vip_address: "172.24.33.58"
network_interface: "ens6"
neutron_external_interface: "dummy0"    # critical: never use management NIC
enable_haproxy: "no"
nova_compute_virt_type: "qemu"
enable_cinder: "yes"
enable_cinder_backend_lvm: "yes"
cinder_volume_group: "cinder-volumes"
```

### Deploy sequence

```bash
kolla-ansible bootstrap-servers -i /etc/kolla/all-in-one
kolla-ansible prechecks     -i /etc/kolla/all-in-one
kolla-ansible deploy        -i /etc/kolla/all-in-one
kolla-ansible post-deploy   -i /etc/kolla/all-in-one
source /etc/kolla/admin-openrc.sh
```

---

## Engineering challenges solved

| # | Challenge | Root cause | Resolution |
|---|---|---|---|
| 1 | Kolla-Ansible CLI syntax failure | Version 20.x changed to collection model | Pinned to 18.4.0 + ansible-core 2.15 |
| 2 | `stable/2024.1` branch not found | Branch moved to `unmaintained/2024.1` | Patched `requirements.yml` |
| 3 | `docker-compose-plugin` dpkg conflict | Ubuntu `docker-compose-v2` package clash | Removed conflicting Ubuntu package |
| 4 | `dbus-python` build failure | Missing `pkg-config` + `libdbus-1-dev` | Installed system build deps |
| 5 | Management NIC lockout during deploy | OVS enslaved `ens6` into `br-ex` | Used `dummy0` as `neutron_external_interface` |
| 6 | MariaDB VIP connection failure | `kolla_internal_vip_address` was commented out | Uncommented and set to management IP |
| 7 | `openvswitch_db` container crash | System OVS pidfile conflicting with containerised OVS | Disabled system `openvswitch-switch` service |
| 8 | VM ERROR on launch | Host lacks KVM extensions (nested VM) | Switched `nova_compute_virt_type` to `qemu` |
| 9 | Floating IP unreachable | `br-ex` had no physical backing (dummy0) | Added `ens6` as OVS port on `br-ex` |
| 10 | CORS on dashboard API calls | Browser calling OpenStack APIs directly | Node.js proxy layer; Nginx `/api/` passthrough |

---

## Live environment

```bash
# Verify full stack
source /etc/kolla/admin-openrc.sh
openstack server list
openstack network list
openstack volume list
openstack image list

# Custom dashboard API
curl http://localhost:9000/api/instances
curl http://localhost:9000/api/quotas
curl http://localhost:9000/api/hypervisors

# Container health
docker ps --format "table {{.Names}}\t{{.Status}}" | grep -v healthy
```

---

## Project structure

```
/etc/kolla/                          Kolla-Ansible config
  globals.yml                        Deployment configuration
  passwords.yml                      Service credentials (gitignored)
  admin-openrc.sh                    OpenStack CLI credentials

/opt/kolla-venv/                     Python virtualenv
  bin/kolla-ansible                  Deployment CLI

/opt/infraops-dashboard/
  backend/
    server.js                        Express proxy server
    .env                             OpenStack credentials (gitignored)
    package.json
  frontend/
    src/
      Dashboard.jsx                  Main React application
      main.jsx
    dist/                            Production build (served by Nginx)
    package.json
    vite.config.js

/etc/nginx/sites-available/
  infraops-dashboard                 Nginx site config (port 9000)

/etc/systemd/system/
  infraops-backend.service           Backend proxy systemd unit
```

---

## Skills demonstrated

- **Infrastructure as Code** — Kolla-Ansible playbooks, globals.yml tuning, inventory management
- **Container orchestration** — 30+ Docker containers, lifecycle management, inter-service dependencies
- **Cloud networking** — Open vSwitch, Neutron ML2, provider/tenant network isolation, floating IPs, OVS bridge management
- **Block storage** — LVM physical volume setup, Cinder volume group, attach/detach lifecycle
- **Linux systems engineering** — systemd services, netplan, kernel module management, dpkg conflict resolution
- **API integration** — Keystone token auth, multi-service REST orchestration, token caching
- **Full-stack development** — React 18, Express.js, Nginx reverse proxy, Vite build pipeline
- **Troubleshooting** — 10+ production-class issues diagnosed and resolved under real constraints

---

## Author

**Derik (Dereje Sichala)**
DevOps Engineer & Cloud Infrastructure Engineer
4+ years enterprise infrastructure · Oracle Exadata · Nutanix HCI · OpenStack

---

*Built as a portfolio infrastructure lab. All services running on production-grade hardware in Addis Ababa, Ethiopia.*
