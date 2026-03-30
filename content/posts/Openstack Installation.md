+++
title = "a definitive guide for OpenStack Installation"
date = 2026-01-20
[taxonomies]
tags = ["openstack"]

[extra]
mermaid = true
+++


### Network Setup
##### Openstack Network connections

{% mermaid() %}
graph LR

%% =============================
%% Router and Switch
%% =============================

OPN[OPNsense Router]
SW[Core Switch VLAN Trunk]

OPN --- SW

%% =============================
%% Controller Node
%% =============================

subgraph Controller
  CMGMT[Mgmt 10.88.10.80]
  CTUN[VLAN212 10.88.20.80]
  CEXTAPI[VLAN214 10.88.40.80]
  CSTOR[Storage 192.168.20.80]
  CINTVIP[Internal VIP 10.88.10.100]
  CEXTVIP[External VIP 10.88.40.100]
end

%% =============================
%% Compute1 Node
%% =============================

subgraph Compute1
  C1MGMT[Mgmt 10.88.10.90]
  C1TUN[VLAN212 10.88.20.90]
  C1EXT[VLAN214 10.88.40.90]
  C1STOR[Storage 192.168.20.90]
end

%% =============================
%% Compute2 Node
%% =============================

subgraph Compute2
  C2MGMT[Mgmt 10.88.10.91]
  C2TUN[VLAN212 10.88.20.91]
  C2EXT[VLAN214 10.88.40.91]
  C2STOR[Storage 192.168.20.91]
end

%% =============================
%% Logical Networks
%% =============================

subgraph Management_Network
  MGMTNET[10.88.10.0/24]
end

subgraph Tunnel_Network_VLAN212
  TUNNET[10.88.20.0/24 Geneve]
end

subgraph External_API_VLAN214
  EXTAPINET[10.88.40.0/24]
end

subgraph Provider_Network_VLAN213
  PROVIDER[physnet1 br-ex]
end

subgraph Storage_Network
  STORNET[192.168.20.0/24 Ceph]
end

%% =============================
%% Connections
%% =============================

SW --- CMGMT
SW --- CTUN
SW --- CEXTAPI
SW --- C1TUN
SW --- C2TUN
SW --- C1EXT
SW --- C2EXT

CMGMT --- MGMTNET
C1MGMT --- MGMTNET
C2MGMT --- MGMTNET
CINTVIP --- MGMTNET

CTUN --- TUNNET
C1TUN --- TUNNET
C2TUN --- TUNNET

CEXTAPI --- EXTAPINET
C1EXT --- EXTAPINET
C2EXT --- EXTAPINET
CEXTVIP --- EXTAPINET

CSTOR --- STORNET
C1STOR --- STORNET
C2STOR --- STORNET

PROVIDER --- SW

{% end %}

- Physical wiring: each server has a trunk (trunk0 / eno2 / similar) carrying all VLANs (212, 213, 214). The switch is VLAN-aware and tags traffic per VLAN to each host.
  
- Management network (10.88.10.0/24) is used for internal OpenStack service communication (API traffic, kolla_internal_vip).
  
- VLAN 212 (10.88.20.0/24) is reserved for overlay tunnels (VXLAN/Geneve) that OVN/Neutron uses for tenant networking between compute nodes.
  
- VLAN 213 is the provider/external network (physnet1 / br-ex). This is where floating IPs and external provider networks live.
  
- VLAN 214 (10.88.40.0/24) is used for external API access and the external VIP (kolla_external_vip 10.88.40.100) which front-ends Horizon / public APIs via HAProxy + Keepalived.
  
- Storage (192.168.20.0/24) is an isolated network for Ceph / block storage traffic (separate NICs: stor0 / ens2f1).
  
- HA/Load balancing: HAProxy + Keepalived run on controller(s) and use the external VIP for public API access; internal VIP handles management failover.

#### Layer 2 Physical VLAN Trunk Diagram
{% mermaid() %}
graph LR

%% ================================
%% Core Switch
%% ================================

SW[Core Switch VLAN Aware]

%% ================================
%% OPNsense
%% ================================

subgraph OPNsense
  OPN_TRUNK[Trunk Port]
  OPN_V212[VLAN212]
  OPN_V213[VLAN213]
  OPN_V214[VLAN214]
end

%% ================================
%% Controller
%% ================================

subgraph Controller
  C_TRUNK[trunk0]
  C_V212[vlan212]
  C_V213[vlan213 br-ex]
  C_V214[vlan214]
  C_MGMT[mgmt 10.88.10.80]
  C_STOR[stor0 192.168.20.80]
end

%% ================================
%% Compute1
%% ================================

subgraph Compute1
  C1_TRUNK[eno2]
  C1_V212[vlan212]
  C1_V213[vlan213 br-ex]
  C1_V214[vlan214]
  C1_MGMT[eno1 10.88.10.90]
  C1_STOR[ens2f1 192.168.20.90]
end

%% ================================
%% Compute2
%% ================================

subgraph Compute2
  C2_TRUNK[trunk0]
  C2_V212[vlan212]
  C2_V213[vlan213 br-ex]
  C2_V214[vlan214]
  C2_MGMT[mgmt 10.88.10.91]
  C2_STOR[stor0 192.168.20.91]
end

%% ================================
%% Trunk Links (Tagged VLANs)
%% ================================

SW ---|Tagged 212,213,214| C_TRUNK
SW ---|Tagged 212,213,214| C1_TRUNK
SW ---|Tagged 212,213,214| C2_TRUNK
SW ---|Tagged 212,213,214| OPN_TRUNK

%% ================================
%% VLAN Subinterfaces
%% ================================

C_TRUNK --> C_V212
C_TRUNK --> C_V213
C_TRUNK --> C_V214

C1_TRUNK --> C1_V212
C1_TRUNK --> C1_V213
C1_TRUNK --> C1_V214

C2_TRUNK --> C2_V212
C2_TRUNK --> C2_V213
C2_TRUNK --> C2_V214

OPN_TRUNK --> OPN_V212
OPN_TRUNK --> OPN_V213
OPN_TRUNK --> OPN_V214

%% ================================
%% Separate Access Ports
%% ================================

SW ---|Access VLAN Mgmt| C_MGMT
SW ---|Access VLAN Mgmt| C1_MGMT
SW ---|Access VLAN Mgmt| C2_MGMT

SW ---|Access VLAN Storage| C_STOR
SW ---|Access VLAN Storage| C1_STOR
SW ---|Access VLAN Storage| C2_STOR

{% end %}

#### Trunk Ports (Tagged)

These carry multiple VLANs:

- VLAN 212 → Tunnel (Geneve overlay)
- VLAN 213 → Provider / br-ex
- VLAN 214 → External API


Trunk interfaces:

- Controller → `trunk0`
- Compute1 → `eno2`
- Compute2 → `trunk0`
- OPNsense → VLAN trunk port


All are **802.1Q tagged**.

---

#### Access Ports (Untagged)

### Management Network

- 10.88.10.0/24
- Connected via mgmt / eno1

- Used for:
    - kolla_internal_vip
    - API communication
    - SSH / Ansible

### Storage Network

- 192.168.20.0/24
- Dedicated Ceph backend traffic
- Isolated from tenant traffic


#### Layer 3 Routing Diagram (With OPNsense Interfaces)
{% mermaid() %}
graph LR

%% =========================
%% Internet
%% =========================

INTERNET[Internet]

%% =========================
%% OPNsense Router
%% =========================

subgraph OPNsense
  WAN[WAN]
  MGMTGW[Mgmt GW 10.88.10.1]
  V212[VLAN212 GW]
  V213[VLAN213 Provider GW]
  V214[VLAN214 GW 10.88.40.1]
end

INTERNET --> WAN

%% =========================
%% Management Network
%% =========================

subgraph Management_10_88_10_0_24
  CTRL_M[Controller 10.88.10.80]
  CMP1_M[Compute1 10.88.10.90]
  CMP2_M[Compute2 10.88.10.91]
  INT_VIP[Internal VIP 10.88.10.100]
end

MGMTGW --> CTRL_M
MGMTGW --> CMP1_M
MGMTGW --> CMP2_M
MGMTGW --> INT_VIP

%% =========================
%% Tunnel Network VLAN212
%% =========================

subgraph Tunnel_10_88_20_0_24
  CTRL_T[Controller 10.88.20.80]
  CMP1_T[Compute1 10.88.20.90]
  CMP2_T[Compute2 10.88.20.91]
end

V212 --> CTRL_T
V212 --> CMP1_T
V212 --> CMP2_T

%% =========================
%% External API VLAN214
%% =========================

subgraph External_API_10_88_40_0_24
  CTRL_E[Controller 10.88.40.80]
  CMP1_E[Compute1 10.88.40.90]
  CMP2_E[Compute2 10.88.40.91]
  EXT_VIP[External VIP 10.88.40.100]
end

V214 --> CTRL_E
V214 --> CMP1_E
V214 --> CMP2_E
V214 --> EXT_VIP

%% =========================
%% Provider Network VLAN213
%% =========================

subgraph Provider_physnet1
  FIP[Floating IP Pool]
  VM[VM Instance]
end

V213 --> FIP
FIP --> VM

%% =========================
%% Storage Network
%% =========================

subgraph Storage_192_168_20_0_24
  CTRL_S[Controller 192.168.20.80]
  CMP1_S[Compute1 192.168.20.90]
  CMP2_S[Compute2 192.168.20.91]
end

CTRL_S --- CMP1_S
CTRL_S --- CMP2_S

%% =========================
%% WAN Routing
%% =========================

WAN --> MGMTGW
WAN --> V214
WAN --> V213

{% end %}

#### OVN Packet Walk Diagram

{% mermaid() %}
flowchart TD

%% =========================
%% Internet
%% =========================

INTERNET[Internet Client]

%% =========================
%% OPNsense
%% =========================

OPN_WAN[OPNsense WAN]
OPN_V213[VLAN213 Gateway]

INTERNET --> OPN_WAN
OPN_WAN --> OPN_V213

%% =========================
%% Compute Node Physical
%% =========================

BR_EX[br-ex on Compute1]
PATCH_EX_INT[patch-ex to br-int]

OPN_V213 --> BR_EX
BR_EX --> PATCH_EX_INT

%% =========================
%% Integration Bridge
%% =========================

BR_INT[br-int]
OVN_ROUTER[OVN Logical Router]
DNAT[DNAT FloatingIP 203.0.113.50 -> 192.168.100.10]

PATCH_EX_INT --> BR_INT
BR_INT --> OVN_ROUTER
OVN_ROUTER --> DNAT

%% =========================
%% VM Network
%% =========================

VM_PORT[VM vNIC Port]
VM[VM 192.168.100.10]

DNAT --> VM_PORT
VM_PORT --> VM

{% end %}

#### Controller netplan from Ubuntu 24.04 LTS
```
network:
  version: 2
  ethernets:
    mgmt:
      addresses:
        - 10.88.10.80/24
      routes:
        - to: default
          via: 10.88.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
    trunk0:
      dhcp4: false
      dhcp6: false
    stor0:
      addresses:
        - 192.168.20.80/24
  vlans:
    vlan212:
      id: 212
      link: trunk0
      addresses:
        - 10.88.20.80/24
    vlan213:
      id: 213
      link: trunk0
    vlan214:
      id: 214
      link: trunk0
      addresses:
        - 10.88.40.80/24
```

#### Compute 1 netplan
```
network:
  version: 2
  ethernets:
    eno1:
      addresses:
        - 10.88.10.90/24
      routes:
        - to: default
          via: 10.88.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
    eno2:
      dhcp4: false
      dhcp6: false
    ens2f1:
      addresses:
        - 192.168.20.90/24
  vlans:
    vlan212:
      id: 212
      link: eno2
      addresses:
        - 10.88.20.90/24
    vlan213:
      id: 213
      link: eno2
    vlan214:
      id: 214
      link: eno2
      addresses:
        - 10.88.40.90/24
```

#### Compute 2 netplan
```
network:
  version: 2

  ethernets:
    mgmt:
      addresses:
        - 10.88.10.91/24
      routes:
        - to: default
          via: 10.88.10.1
      nameservers:
        addresses:
          - 8.8.8.8
          - 1.1.1.1
    trunk0:
      dhcp4: false
      dhcp6: false
    stor0:
      addresses:
        - 192.168.20.91/24
  vlans:
    vlan212:
      id: 212
      link: trunk0
      addresses:
        - 10.88.20.91/24
    vlan213:
      id: 213
      link: trunk0
    vlan214:
      id: 214
      link: trunk0
      addresses:
        - 10.88.40.91/24
```

#### Explanation of `kolla/globals.yml` 

- `kolla_base_distro: "ubuntu"` — use Ubuntu base images for containers
- `kolla_internal_vip_address: "10.88.10.100"` — VIP for internal management endpoint (keystone, db clients etc.).
- `kolla_external_vip_address: "10.88.40.100"` — VIP for public APIs and Horizon (fronted by HAProxy + Keepalived).
- `kolla_external_vip_interface: "vlan214"` — the external VIP sits on VLAN214(10.88.40.90/24).
- `network_interface: "mgmt"` — host interface used for management traffic.
- `neutron_external_interface: "vlan213"` — the physical interface bridged into `br-ex`.
- `tunnel_interface: "vlan212"` — the VLAN used to carry overlay tunnels between compute nodes.

**OpenStack services & compute**

- `enable_cinder: "yes"` — deploy Cinder (Block Storage).
- `nova_compute_virt_type: "kvm"` — use KVM on compute nodes.
- `neutron_plugin_agent: "ovn"` — OVN (Open Virtual Network) as Neutron backend (recommended for distributed services) *kolla-ansible recently changed to `ovn` from `ovs` since 25.2.*
- `neutron_ovn_distributed_fip: "yes"` — enable distributed floating IPs so NAT happens at compute nodes (removes central gateway bottlenecks).
- `neutron_ovn_dhcp_agent: "yes"` — OVN DHCP agent for handing out in-VM leases.
- `enable_ovn_db_cluster: "yes"` — set up OVN northbound/southbound DB HA (useful for multi-controller HA).
- `enable_neutron_agent_ovs: "no"` — not using classic OVS neutron agent (OVN replaces most parts).

**Coordination / key-value store**

- `enable_redis: "yes" ` New version 2025.2 doesnot support redis instead it supports valkey. ***Kolla 2025.2 introduces Valkey** (a replacement role and mechanism so deployments can use Valkey instead of Redis; Kolla provides a Valkey role and a migration path). If you’re deploying 2025.2, prefer following the release notes about Valkey and `kolla-genpwd`/`kolla-mergepwd` changes.* ([OpenStack Docs](https://docs.openstack.org/releasenotes/kolla-ansible/2025.2.html?utm_source=chatgpt.com "2025.2 Series Release Notes — Kolla Ansible Release Notes ..."))

**HA / load balancing / extras**

- `enable_haproxy: "yes"` & `enable_keepalived: "yes"` — HAProxy + Keepalived provide the public and internal VIPs for APIs and Horizon.
- `neutron_bridge_name: "br-ex"` — the external bridge name used by Kolla for provider networks.
- `neutron_physical_network: "physnet1"` — maps the `br-ex` bridge to this logical physical network.
- `neutron_type_drivers: "flat,vlan,vxlan,geneve"` — supported Neutron network types; we're allowing many types but explicitly using `geneve` for tenant networks.
- `neutron_tenant_network_types: "geneve"` — tenant overlay = Geneve.

**Storage / Ceph integration**

- `enable_ceph: "yes"` + `glance_backend_ceph: "yes"` + `cinder_backend_ceph: "yes"` + `nova_backend_ceph: "yes"` — these tell Kolla to configure services to use Ceph (RBD) for images, volumes, and ephemeral disks. **Important**: Kolla **does not provision** the Ceph cluster for you; you must provision Ceph (with `cephadm` or ansible) and then provide pools, keyrings, and the Ceph config to Kolla which we'll see in deployment steps. ([OpenStack Docs](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html?utm_source=chatgpt.com "External Ceph — kolla-ansible 21.1.0.dev411 documentation"))
- `ceph_*` variables (`ceph_glance_user`, `ceph_cinder_user`, `ceph_nova_user`, `ceph_*_pool_name`, `*_keyring`) — used by Kolla to populate container config pointing to Ceph pools and keyrings you created.

**Backup & high-availability helpers**

- `enable_mariabackup: "yes"` — enable MariaDB backup tool integration.
  
- `enable_masakari: "yes"` — Masakari provides instance HA (automatic failover/restart of instances on host failures). Useful for production HA scenarios.
  
- `enable_skyline: "yes"` — enable the Skyline modern dashboard (if you want a modern UI instead of or alongside Horizon). Kolla includes a `skyline` role and configuration; enabling it will deploy Skyline containers and wire it to Keystone & Prometheus if configured. ([OpenStack Docs](https://docs.openstack.org/kolla-ansible/latest/reference/shared-services/skyline-guide.html?utm_source=chatgpt.com "Skyline OpenStack dashboard — kolla-ansible 21.1.0. ..."))
  

#### Install flow 

**Important**: Kolla 2025.2 includes changes (Valkey migration from Redis, OVN improvements). Read the 2025.2 release notes and follow migration steps if upgrading. ([OpenStack Docs](https://docs.openstack.org/releasenotes/kolla-ansible/2025.2.html?utm_source=chatgpt.com "2025.2 Series Release Notes — Kolla Ansible Release Notes ..."))

**High-level steps**

1. Prepare OS on all hosts (matching `kolla_base_distro` and kernel/iptables/etc).
   
2. Ensure DNS / hostnames / NTP are correct.
   
3. Create `inventory` mapping (controller & compute groups) and set `globals.yml` and `passwords.yml`.
   
4. Provision Ceph first (external to Kolla) — we’ll use `cephadm` here. Kolla expects pools and keyrings to exist. ([OpenStack Docs](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html?utm_source=chatgpt.com "External Ceph — kolla-ansible 21.1.0.dev411 documentation"))
   
5. Put Ceph config + keyrings into `/etc/kolla/config/*` so Kolla containers can consume them.
   
6. Run Kolla prechecks and deploy.
   
7. Post-deploy tests: `openstack` client calls (create network, boot VM, attach volume, floating IP test), and Ceph validations.
   

---

## 5) **Ceph provisioning** (use these exact commands in your environment)

> Kolla expects Ceph (external) to be provisioned and pools/keyrings created. For small setups `cephadm` is the recommended orchestrator.

Run on your Ceph bootstrap host (use the host’s **management/internal IP** for `--mon-ip` and `--cluster-network`):

```bash
# Install cephadm from Ubuntu Ceph Squid Repo
apt-get install cephadm -y

# Bootstrap the Ceph cluster (Monitor IP is the mgmt/internal IP)
cephadm bootstrap \
    --mon-ip {get ip from above network}\
    --single-host-defaults \
    --cluster-network={get ip from above network} \
    --initial-dashboard-password=startsm \
    --dashboard-password-noupdate \
    --allow-fqdn-hostname | tee cephadm-bootstrap.log
```

Then:

```bash
# Access Ceph Shell
cephadm shell
# Check cluster status
ceph -s
# Exit shell
exit

# Install client tools (outside of cephadm shell)
apt install ceph-common

ceph orch apply osd --all-available-devices

ceph osd pool create volumes
ceph osd pool create images
ceph osd pool create backups
ceph osd pool create vms

# Set replication size to 1 for the AIO single-node deployment
ceph config set global mon_allow_pool_size_one true
ceph osd pool set volumes size 1 --yes-i-really-mean-it
ceph osd pool set .mgr size 1 --yes-i-really-mean-it # For the mgr metadata pool
ceph osd pool set images size 1 --yes-i-really-mean-it
ceph osd pool set backups size 1 --yes-i-really-mean-it
ceph osd pool set vms size 1 --yes-i-really-mean-it

# Enable the RBD application on the new pools
ceph osd pool application enable volumes rbd
ceph osd pool application enable images rbd
ceph osd pool application enable backups rbd
ceph osd pool application enable vms rbd

# Optional: Mute the expected health warning for single-node setup
ceph health mute POOL_NO_REDUNDANCY

#Validate
ceph -s
ceph osd lspools
ceph osd tree

# Create Ceph Users and Keyrings (Output to /etc/ceph/)
ceph auth get-or-create client.glance mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=images' -o /etc/ceph/ceph.client.glance.keyring
ceph auth get-or-create client.cinder mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=volumes, allow rwx pool=images' -o /etc/ceph/ceph.client.cinder.keyring
ceph auth get-or-create client.cinder-backup mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=backups' -o /etc/ceph/ceph.client.cinder-backup.keyring
ceph auth get-or-create client.nova mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rx pool=images' -o /etc/ceph/ceph.client.nova.keyring
```

> Note: `cephadm` is the supported lifecycle manager for Ceph and is the recommended path to create the Ceph cluster when using Kolla external Ceph. ([Ceph Documentation](https://docs.ceph.com/en/reef/cephadm/?utm_source=chatgpt.com "Cephadm - Ceph Documentation"))

### Copy Ceph configs/keyrings into Kolla config tree

(so containers can authenticate to Ceph):

```bash
# Generate SSL certificates for Kolla (if you want TLS for APIs)
kolla-ansible certificates -i all-in-one

# Create config directories for services that will use Ceph
mkdir /etc/kolla/config
mkdir /etc/kolla/config/nova
mkdir /etc/kolla/config/glance
mkdir -p /etc/kolla/config/cinder/cinder-volume
mkdir /etc/kolla/config/cinder/cinder-backup

# Copy Ceph configuration files
cp /etc/ceph/ceph.conf /etc/kolla/config/cinder/
cp /etc/ceph/ceph.conf /etc/kolla/config/nova/
cp /etc/ceph/ceph.conf /etc/kolla/config/glance/

# Copy Ceph Keyrings
cp /etc/ceph/ceph.client.glance.keyring /etc/kolla/config/glance/
cp /etc/ceph/ceph.client.nova.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/nova/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-volume/
cp /etc/ceph/ceph.client.cinder.keyring /etc/kolla/config/cinder/cinder-backup/
cp /etc/ceph/ceph.client.cinder-backup.keyring /etc/kolla/config/cinder/cinder-backup/
```

---

## 6) Kolla Ansible deployment steps (control host)

**Preparatory steps (one-time on a control machine)**

```bash
# Install required dependecies(Ubuntu used here, can use other distros as well)
sudo apt install git python3-dev libffi-dev gcc libssl-dev libdbus-glib-1-dev

# Create python virtual environment
python3 -m venv .venv
source .venv/bin/activate

# update pip (good habit)
pip install -U pip

# Install kolla-ansible repo(I'm installing 2025.2 version)
pip install git+https://opendev.org/openstack/kolla-ansible@stable/2025.2

# Create /etc/koll directory with permissions
sudo mkdir -p /etc/kolla
sudo chown $USER:$USER /etc/kolla

# Copy globals.yml and passwords.yml to /etc/kolla directory 
cp -r .venv/share/kolla-ansible/etc_examples/kolla/* /etc/kolla
 
# Copy `all-in-one` inventory file to the current directory
cp .venv/share/kolla-ansible/ansible/inventory/all-in-one .

# Install Ansible galaxy dependecies
kolla-ansible install-deps

# Install python deps
pip install docker dbus-python
```

**Configure `/etc/kolla/globals.yml`** — paste your `globals` values (the block you provided). Make sure IPs, interfaces and VLANs match your netplan.

**Create `/etc/kolla/passwords.yml`** (it’s shipped but usually blank). Generate passwords:

```bash
# generate passwords for Kolla
kolla-genpwd -p /etc/kolla/passwords.yml
```

(See Kolla tools `kolla-genpwd` and `kolla-mergepwd` for upgrades or merging with existing installs). ([OpenDev: Free Software Needs Free Tools](https://opendev.org/openstack/kolla-ansible/src/branch/master/setup.cfg?utm_source=chatgpt.com "kolla-ansible/setup.cfg at master"))

**Inventory** — create an Ansible inventory, e.g.:

`/etc/kolla/inventory/all-inventory` with groups (controller, compute) and `ansible_host` entries for each node (or follow the kolla documentation example inventory).

**Pre-flight checks:**

```bash
# Run kolla-ansible prechecks
kolla-ansible -i /etc/kolla/inventory/all-inventory prechecks
```

(Prechecks will validate SSH connectivity, kernel/sysctl, container runtime availability, etc.) ([OpenStack Docs](https://docs.openstack.org/kolla/mitaka/quickstart.html?utm_source=chatgpt.com "Deployment of Kolla on Bare Metal or Virtual Machine"))

**Generate configuration & deploy:**

```bash
# generate config files for enabled services
kolla-ansible -i /etc/kolla/inventory/all-inventory genconfig

# deploy (this pulls images, creates containers, configures services)
kolla-ansible -i /etc/kolla/inventory/all-inventory deploy
```

If you change configuration after deploy, use:

```bash
# to update config but not restart containers:
kolla-ansible -i /etc/kolla/inventory/all-inventory genconfig

# to reconfigure/restart containers:
kolla-ansible -i /etc/kolla/inventory/all-inventory reconfigure
```

**Post-deploy** — Source the admin environment and validate APIs:

```bash
# Source admin credentials (generated by Kolla)
source /etc/kolla/admin-openrc.sh

# Check keystone / list services
openstack service list
```

(If Keystone endpoint isn’t available, check HAProxy + Keepalived / VIPs.)

---

## 7) Concrete smoke tests (what to run first)

Source admin credentials (on the control host or a host that has the client tools):

```bash
source /etc/kolla/admin-openrc.sh
openstack project list
openstack user list
openstack service list
```

### Networking sanity

```bash
# List neutron agents
openstack network agent list

# List networks
openstack network list
```

### Image + Compute + Keypair

```bash
# Upload a tiny cirros or ubuntu image
openstack image create cirros --file cirros-0.5.2-x86_64-disk.img --disk-format qcow2 --container-format bare --public

# Create flavor
openstack flavor create --ram 512 --vcpus 1 --disk 1 m1.tiny

# Create keypair
openstack keypair create demo-key > demo-key.pem
chmod 600 demo-key.pem

# Create a network, subnet and router (tenant network)
openstack network create demo-net
openstack subnet create --network demo-net --subnet-range 192.168.100.0/24 demo-subnet
openstack router create demo-router
openstack router add subnet demo-router demo-subnet
openstack router set --external-gateway <public-network> demo-router  # <public-network> is your provider network name

# Boot instance
openstack server create --image cirros --flavor m1.tiny --key-name demo-key --network demo-net test-vm

# Check instance
openstack server list
```

### Volume (Cinder) + Attach + Ceph validation

```bash
# Create a volume (this should create a Ceph RBD image if cinder->ceph is configured)
openstack volume create --size 1 test-volume
openstack volume list

# Attach to server
openstack server add volume test-vm test-volume

# Verify Ceph pools & RBD images on ceph cluster:
ceph -s
ceph osd lspools
# On Ceph side: rbd ls volumes  (or use cephadm shell then rbd)
```

### Floating IP test (external reachability)

```bash
# Allocate and assign a floating IP (from provider network)
openstack floating ip create <public-network>
openstack server add floating ip test-vm <floating-ip>

# From the internet (or an external host that can route to VLAN213), curl the public IP
curl http://<floating-ip>
```

If `neutron_ovn_distributed_fip: "yes"` is set, NAT is distributed at compute and you'll see DNAT/SNAT handled in the compute host (no centralized NAT gateway).

---

## 8) Troubleshooting checklist (quick)

- **VIPs not present**: check `keepalived` containers and the host’s IP table/VRRP state.
  
- **Images failing to upload / Glance errors**: check Ceph keyring presence in `/etc/kolla/config/glance/` and container logs for `glance-api`.
  
- **Cinder volume creation stuck**: check `cinder-volume` logs and Ceph OSD health with `ceph -s`.
  
- **OVN/Neutron connectivity issues**: verify `ovn-controller` running on computes and `ovn-northd`/DB on controllers (Kolla 2025.2 added improvements to OVN container envs to make `ovn-nbctl`/`ovn-sbctl` manageable). ([OpenStack Docs](https://docs.openstack.org/releasenotes/kolla-ansible/2025.2.html?utm_source=chatgpt.com "2025.2 Series Release Notes — Kolla Ansible Release Notes ..."))
  
- **Redis/Coordination errors**: if you upgraded from older Kolla, check Valkey variables and migration guidance — 2025.2 introduces a Valkey role and sentinel support. ([OpenStack Docs](https://docs.openstack.org/releasenotes/kolla-ansible/2025.2.html?utm_source=chatgpt.com "2025.2 Series Release Notes — Kolla Ansible Release Notes ..."))
  

---

## 9) Notes on upgrades & external Ceph

- Kolla **intentionally does not** provision Ceph for you; use `cephadm`/ceph-ansible and create pools & keyrings. Kolla will use those keyrings/pools once copied into `/etc/kolla/config/*`. ([OpenStack Docs](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html?utm_source=chatgpt.com "External Ceph — kolla-ansible 21.1.0.dev411 documentation"))
  
- If you are upgrading from a release that used Redis, be aware of the Valkey migration changes in 2025.2 and follow the release notes / migration documentation carefully. ([OpenStack Docs](https://docs.openstack.org/releasenotes/kolla-ansible/2025.2.html?utm_source=chatgpt.com "2025.2 Series Release Notes — Kolla Ansible Release Notes ..."))
  

---

## 10) Final tips & checklist before you press `deploy`

- Ensure `/etc/hosts` resolves hostnames used in the inventory (Kolla uses inventory hostnames heavily).
  
- Ensure `ansible_user` has passwordless sudo on managed nodes.
  
- Confirm netplan configuration—management IPs & trunk interfaces must be stable and reachable from the control host.
  
- Copy Ceph `ceph.conf` and keyrings into `/etc/kolla/config/*` exactly as shown above.
  
- Generate passwords with `kolla-genpwd` and keep `/etc/kolla/passwords.yml` safe.
  
- Run `kolla-ansible -i INVENTORY prechecks` and fix everything it reports.
  

---

## Useful links (again)

- Kolla Ansible 2025.2 release notes (Valkey + OVN notes). ([OpenStack Docs](https://docs.openstack.org/releasenotes/kolla-ansible/2025.2.html?utm_source=chatgpt.com "2025.2 Series Release Notes — Kolla Ansible Release Notes ..."))
  
- Kolla external-Ceph guidance. ([OpenStack Docs](https://docs.openstack.org/kolla-ansible/latest/reference/storage/external-ceph-guide.html?utm_source=chatgpt.com "External Ceph — kolla-ansible 21.1.0.dev411 documentation"))
  
- Cephadm docs & bootstrap reference. ([Ceph Documentation](https://docs.ceph.com/en/reef/cephadm/?utm_source=chatgpt.com "Cephadm - Ceph Documentation"))
  
- Kolla operating docs / tools (kolla-genpwd / prechecks / genconfig). ([OpenStack Docs](https://docs.openstack.org/kolla-ansible/latest/user/operating-kolla.html?utm_source=chatgpt.com "Operating Kolla — kolla-ansible 21.1.0.dev416 ..."))
  
- Skyline Dashboard docs (if you enabled Skyline). ([OpenStack Docs](https://docs.openstack.org/kolla-ansible/latest/reference/shared-services/skyline-guide.html?utm_source=chatgpt.com "Skyline OpenStack dashboard — kolla-ansible 21.1.0. ..."))
  
