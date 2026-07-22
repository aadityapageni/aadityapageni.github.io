+++
title = "Compute Node Failover Architecture — vBMC + Pacemaker + Masakari HA"
date = 2026-07-22
[taxonomies]
tags = ["openstack","masakari"]
[extra]
mermaid = true
+++


## Table of Contents

1. [Introduction](#1-introduction)
2. [Architectural Flow](#2-architectural-flow)
   - 2.1 [End-to-End Failure & Recovery Sequence](#21-end-to-end-failure--recovery-sequence)
   - 2.2 [Fencing Sub-Flow Diagram](#22-fencing-sub-flow-diagram)
3. [Prerequisites](#3-prerequisites)
4. [Configuration Guide](#4-configuration-guide)
   - 4.1 [Step 1 — Initialize Virtual BMC on the Physical Hypervisor](#41-step-1--initialize-virtual-bmc-on-the-physical-hypervisor)
   - 4.2 [Step 2 — Install & Configure Corosync on Compute Nodes](#42-step-2--install--configure-corosync-on-compute-nodes)
   - 4.3 [Step 3 — Establish Quorum via QDevice](#43-step-3--establish-quorum-via-qdevice)
   - 4.4 [Step 4 — Configure Pacemaker STONITH to Target vBMC Endpoints](#44-step-4--configure-pacemaker-stonith-to-target-vbmc-endpoints)
   - 4.5 [Step 5 — Configure Kolla-Ansible for Masakari (Instance HA)](#45-step-5--configure-kolla-ansible-for-masakari-instance-ha)
   - 4.6 [Step 6 — Align Masakari Checks for vBMC Compatibility](#46-step-6--align-masakari-checks-for-vbmc-compatibility)
   - 4.7 [Step 7 — Map Masakari Hosts to the OpenStack Dashboard](#47-step-7--map-masakari-hosts-to-the-openstack-dashboard)
5. [Validation & Testing](#5-validation--testing)
   - 5.1 [Manual Fencing Validation](#51-manual-fencing-validation)
   - 5.2 [Full Fencing Test Procedure](#52-full-fencing-test-procedure)
6. [Troubleshooting](#6-troubleshooting)
7. [Recovery — Restoring a Fenced Compute Node](#7-recovery--restoring-a-fenced-compute-node)
8. [Appendix — Reference Log Samples](#8-appendix--reference-log-samples)

---

## 1. Introduction

This document describes the high-availability (HA) architecture that protects OpenStack compute nodes against unplanned host failure. It combines three layers:

- **Pacemaker/Corosync** — cluster membership, quorum, and failure detection at the host level.
- **vBMC (Virtual BMC)** — translates standard IPMI network commands into libvirt API calls, allowing STONITH fencing to work against virtualized compute nodes exactly as it would against physical hardware with a real BMC.
- **Masakari** — OpenStack's instance-HA service, which listens for confirmed host-failure events and automatically evacuates instances to a healthy compute node.

Because vBMC translates standard IPMI network commands into libvirt API calls, the Pacemaker `fence_ipmilan` agent cannot tell the difference between virtual hardware and a physical motherboard. This is what allows the same STONITH tooling used on physical racks to work unmodified in a virtualized/nested compute environment.

---

## 2. Architectural Flow

### 2.1 End-to-End Failure & Recovery Sequence

The table below sequences the complete failure-to-recovery flow in the order it actually executes, from initial failure detection through to Masakari reporting a finished recovery.

| # | Stage | Action |
|---|-------|--------|
| 1 | Failure occurs | Compute node fails (network isolation / crash) |
| 2 | Detection | Pacemaker detects node OFFLINE/UNCLEAN |
| 3 | Fencing initiated | Pacemaker executes STONITH: `fence_ipmilan` → vBMC → `virsh destroy` |
| 4 | Fencing executed | vBMC powers off the compute VM |
| 5 | Host-monitor detection | Masakari hostmonitor detects node offline in Corosync |
| 6 | Notification | Hostmonitor sends `COMPUTE_HOST` notification to the Masakari API |
| 7 | Engine intake | Masakari engine processes the notification |
| 8 | Power-state verification | Engine verifies host is powered off (IPMI check — succeeds because vBMC turned it off) |
| 9 | Service disable | Engine disables `nova-compute` for the failed host |
| 10 | Instance lock | Engine locks instances on the failed host |
| 11 | Evacuation call | Engine calls `nova evacuate` for each instance |
| 12 | Instance rebuild | Instances rebuild on a healthy host (shared Ceph RBD storage) |
| 13 | Instance unlock | Engine unlocks the instances |
| 14 | Completion | Notification status: `finished` |

**Why fencing must come before evacuation:** Steps 3–4 are a hard prerequisite for step 11. If a "failed" host is not actually confirmed dead, evacuating its instances elsewhere while the original host is still running risks the same instance running twice against shared Ceph RBD storage — a split-brain condition. The IPMI power-state check at step 8 is what gives Masakari the confidence to proceed.

### 2.2 Fencing Sub-Flow Diagram

The diagram below expands steps 2–4 above — the portion of the flow that happens entirely within Pacemaker and vBMC, before Masakari is ever notified.

```
   [Pacemaker Cluster (compute-node-01)]
                   │
         (Detects node failure)
                   │
                   ▼
   [Executes fence_ipmilan Command]
                   │
       (Sends UDP 623 IPMI Packet)
                   │
                   ▼
     [vBMC Daemon (on KVM Hypervisor)]
                   │
    (Translates to virsh destroy command)
                   │
                   ▼
   [KVM Hypervisor kills compute-node-02 VM]
```

---

## 3. Prerequisites

- A dedicated **KVM/Libvirt Hypervisor** hosting the two virtual compute node instances.
- The `virtualbmc` python package running on that physical hypervisor machine.
- Firewall rules permitting **UDP port 623** (IPMI over LAN traffic) between the compute nodes and the vBMC server.

---

## 4. Configuration Guide

### 4.1 Step 1 — Initialize Virtual BMC on the Physical Hypervisor

Run these steps on the **underlying KVM host** where your virtual compute nodes are running. Do not run this inside the OpenStack nodes.

```bash
# 1. Install virtualbmc
sudo pip3 install virtualbmc

root@compute2:~# for vm in $(virsh list --all --name); do echo -n "$vm: "; virsh dumpxml "$vm" 2>/dev/null | grep -oP '(?<=<nova:name>).*?(?=</nova:name>)'; done | grep "Compute-1"
instance-00000104: Compute-1
root@compute2:~# for vm in $(virsh list --all --name); do echo -n "$vm: "; virsh dumpxml "$vm" 2>/dev/null | grep -oP '(?<=<nova:name>).*?(?=</nova:name>)'; done | grep "compute-2"
instance-000000ff: compute-2

# 2. Add your compute VMs to the vBMC registry
vbmc add instance-00000104 --port 6230 --username admin --password apple
vbmc add instance-000000ff --port 6231 --username admin --password apple

# 3. Start the virtual BMC endpoints
vbmc start instance-00000104
vbmc start instance-000000ff

# 4. Verify vBMC is listening locally
vbmc list
```

*(Note down the IP address of this hypervisor machine; it will serve as the IPMI target IP inside Pacemaker.)*

#### Verify vBMC responds to IPMI from compute nodes

From **compute1**, test that it can reach the vBMC instances on the controller:

```bash
ipmitool -I lanplus \
  -H 10.88.10.91 \
  -p 6231 \
  -U admin \
  -P apple \
  power status
# Expected: Chassis Power is on

ipmitool -I lanplus \
  -H 192.168.100.10 \
  -p 6232 \
  -U admin \
  -P fence-secret \
  power status
# Expected: Chassis Power is on
```

If either fails, check:

- vBMC is running (`vbmc list` shows Status: running)
- UDP port is open on the controller host (vBMC uses UDP, not TCP)
- Firewall isn't blocking the ports: `ufw allow 6231/udp && ufw allow 6232/udp`

**Do not proceed past this point until both ipmitool checks return `Chassis Power is on`.** A working vBMC is the entire foundation of the fencing test.

### 4.2 Step 2 — Install & Configure Corosync on Compute Nodes

Pacemaker must be initialized directly on the compute node host operating systems (outside Kolla containers) to manage physical fencing.

Run these steps on **BOTH `compute-node-01` and `compute-node-02`**:

```bash
# 1. Install Pacemaker, Corosync, and IPMI fencing utilities
sudo apt update
sudo apt install pacemaker pcs fence-agents corosync-qdevice ipmitool -y

# 2. Start and enable the configuration service
sudo systemctl enable --now pcsd

# 3. Set a matching cluster password for the 'hacluster' system user
echo "hacluster:YourClusterSecretSecurePassword" | sudo chpasswd
```

Next, from **`compute-node-01` ONLY**, authenticate and build the cluster pool:

```bash
# Authenticate the two compute hosts
pcs host auth compute-1 compute-2 -u hacluster -p YourClusterSecretSecurePassword

# Setup and start the corosync cluster
pcs cluster setup compute_cluster compute-1 compute-2 (--force)
pcs cluster start --all
pcs cluster enable --all

pcs status corosync
```

### 4.3 Step 3 — Establish Quorum via QDevice

Because this is a 2-node cluster, losing one node drops your cluster votes to 50%, which disables fencing execution. We use a 3rd arbitrator host to bypass this.

Log into your **External Arbitrator Node** (e.g., `controller-01`):

```bash
# Install and activate the lightweight network arbitrator daemon
sudo apt update && sudo apt install pcs corosync-qnetd -y
sudo systemctl enable --now corosync-qnetd
sudo pcs qdevice setup model net --enable --start
sudo systemctl enable --now pcsd
echo "hacluster:YourClusterSecretSecurePassword" | sudo chpasswd
```

Log back into **BOTH `compute-node-01` and `compute-node-02`** to pull down the client agent:

```bash
sudo apt install corosync-qdevice -y
```

From **`compute-node-01` ONLY**, execute the TLS handshake and cluster link configuration:

```bash
# Link your cluster nodes securely to the external arbitrator
> pcs quorum device add model net algorithm=lms host=10.0.0.110
    Setting up qdevice certificates on nodes...
    compute-2: Succeeded
    compute-1: Succeeded
    Enabling corosync-qdevice...
    compute-2: corosync-qdevice enabled
    compute-1: corosync-qdevice enabled
    Sending updated corosync.conf to nodes...
    compute-1: Succeeded
    compute-2: Succeeded
    compute-1: Corosync configuration reloaded
    Starting corosync-qdevice...
    compute-2: corosync-qdevice started
    compute-1: corosync-qdevice started


# Verify the cluster status reflects 3 active voting entities
> pcs quorum status

    Quorum information
    ------------------
    Date:             Wed Jul  1 20:53:59 2026
    Quorum provider:  corosync_votequorum
    Nodes:            2
    Node ID:          2
    Ring ID:          1.e
    Quorate:          Yes

    Votequorum information
    ----------------------
    Expected votes:   3
    Highest expected: 3
    Total votes:      3
    Quorum:           2  
    Flags:            Quorate Qdevice 

    Membership information
    ----------------------
        Nodeid      Votes    Qdevice Name
            1          1    A,V,NMW compute-1
            2          1    A,V,NMW compute-2 (local)
            0          1            Qdevice
```

### 4.4 Step 4 — Configure Pacemaker STONITH to Target vBMC Endpoints

Log into **`compute-node-01`** (inside your OpenStack cluster) to map your STONITH resources. Instead of using the base IPMI port, specify the individual custom destination ports exposed by vBMC.

```bash
# 1. Create fencing target for Compute Node 01
pcs stonith create fence_compute01 fence_ipmilan \
  ip="10.88.10.91" \
  ipport="6230" \
  username="admin" \
  password="apple" \
  lanplus=1 \
  pcmk_host_list="compute-1" \
  op monitor interval=60s

# 2. Create fencing target for Compute Node 02
pcs stonith create fence_compute02 fence_ipmilan \
  ip="10.88.10.91" \
  ipport="6231" \
  username="admin" \
  password="apple" \
  lanplus=1 \
  pcmk_host_list="compute-2" \
  op monitor interval=60s

# 3. Apply a location constraint (Prevents a node from attempting to fence itself)
pcs constraint location fence_compute01 avoids compute-1
pcs constraint location fence_compute02 avoids compute-2

# 4. Commit global STONITH activation
pcs property set stonith-enabled=true   # enable STONITH fencing in the cluster
pcs property set stonith-action=off    # this disables devices to reboot, instead it shit downs the other device
```

### 4.5 Step 5 — Configure Kolla-Ansible for Masakari (Instance HA)

With physical layer fencing ready, we instruct OpenStack to listen to Pacemaker cluster events and automate VM evacuations.

On your **Kolla Deployer machine**, modify your `/etc/kolla/globals.yml` parameter file:

```yaml
enable_masakari: "yes"
enable_masakari_hostmonitor: "no"
enable_masakari_instancemonitor: "yes"
enable_hacluster: "yes"
```

In the inventory file, disable `hacluster_pacemaker_remote` on compute nodes by removing:

```
[hacluster-remote:children]
compute 						#<- remove this
```

Install masakari-hostmonitor natively:

```bash
pip install masakari-monitors --break-system-packages --ignore-installed

pip install --upgrade pyOpenSSL cryptography --break-system-packages

python3 -c "from OpenSSL import SSL; print('ok')"
```

- Configure Masakari Host Monitor with API credentials, Pacemaker/Corosync settings, and logging:

`/etc/masakarimonitors/masakarimonitors.conf`:


```ini
[DEFAULT]
debug = False
log_dir = /var/log/kolla/masakari

[api]
region = RegionOne
auth_url = http://10.0.0.100:5000
user_domain_id = default
project_name = service
project_domain_id = default
username = masakari
password = W6OE7FAoqxtYzzskyQ4NeEUbhBqsx3JfFXFW031s
cafile =
api_interface = internal

[host]
restrict_to_remotes = false
disable_ipmi_check = true
pacemaker_node_type = cluster
corosync_multicast_interfaces = ens3
corosync_multicast_ports = 5405

[oslo_concurrency]
lock_path = /var/lib/masakari/tmp
```

`/etc/systemd/system/masakari-hostmonitor.service`:

```ini
[Unit]
Description=Masakari Host Monitor
After=network-online.target corosync.service pacemaker.service
Wants=network-online.target
Requires=corosync.service pacemaker.service

[Service]
Type=simple
User=root
ExecStart=/usr/local/bin/masakari-hostmonitor \
  --config-file=/etc/masakarimonitors/masakarimonitors.conf
Restart=on-failure
RestartSec=10
StandardOutput=append:/var/log/kolla/masakari/masakari-hostmonitor.log
StandardError=append:/var/log/kolla/masakari/masakari-hostmonitor.log

[Install]
WantedBy=multi-user.target
```

Enable and verify the service:

```bash
mkdir -p /var/log/kolla/masakari
systemctl daemon-reload
systemctl enable --now masakari-hostmonitor
systemctl status masakari-hostmonitor
journalctl -u masakari-hostmonitor -f
```

### 4.6 Step 6 — Align Masakari Checks for vBMC Compatibility

Masakari verifies power statuses via the native tool `ipmitool`. Open your `/etc/kolla/config/masakari.conf` custom configuration override file on your deployment machine and include the port parameters so Masakari maps to the vBMC framework correctly:

```ini
[host]
monitoring_driver = default
disable_ipmi_check = False
ipmi_timeout = 15

# Custom overrides to force ipmitool to find vBMC instances over unique ports
# You can leave this out if you use unique IPs for vBMC instances rather than unique ports.
```

*Note: If both vBMC endpoints share a single hypervisor host IP, ensure your Masakari monitoring configurations can query the custom ports (`6230`/`6231`). If using a flat network map with discrete virtual IPs mapped per vBMC node, the standard default port definitions suffice.*

Re-run the deployment playbook to finalize the infrastructure container logic:

```bash
kolla-ansible -i /path/to/inventory/multinode deploy
```

### 4.7 Step 7 — Map Masakari Hosts to the OpenStack Dashboard

For Masakari to manage failures, the compute nodes must be explicitly registered inside the Masakari API segment. Run these sourcing commands inside your OpenStack CLI environment:

```bash
# Source openstack administrative credentials
source /etc/kolla/admin-openrc.sh

# 1. Form a high-availability segment pool
openstack ha segment create HA-Compute-Segment --recovery_method auto

# 2. Link Compute Node 01 to the segment
openstack ha host create \
  --type COMPUTE \
  --control_attributes SSH \
  --on_maintenance False \
  HA-Compute-Segment compute-node-01

# 3. Link Compute Node 02 to the segment
openstack ha host create \
  --type COMPUTE \
  --control_attributes SSH \
  --on_maintenance False \
  HA-Compute-Segment compute-node-02
```

---

## 5. Validation & Testing

### 5.1 Manual Fencing Validation

Verify the setup manually by triggering a fence execution command directly from the cluster terminal interface:

```bash
# Force-fence compute-node-02 from compute-node-01
pcs stonith fence compute-02
```

**What to expect:**

1. Pacemaker hits `fence_ipmilan` targeting `<HYPERVISOR_HOST_IP>:6231`.
2. The `vbmc` engine handles the payload and invokes a local `virsh destroy compute-node-02` instruction against the VM container on the host.
3. The virtual instance of `compute-node-02` immediately powers off, forcing the Masakari pipeline to safely begin VM evacuations onto `compute-node-01`.

### 5.2 Full Fencing Test Procedure

Everything is in place. Now break compute2 and watch the full flow.

#### 5.2.1 Pre-test setup

Open three terminal windows before starting:

```bash
# Terminal 1 — masakari-engine logs (controller)
docker logs -f masakari_engine

# Terminal 2 — Pacemaker fencing log (compute1)
tail -f /var/log/pacemaker/pacemaker.log | grep -i -E "fence|stonith|ipmi"

# Terminal 3 — cluster status polling (compute1)
watch -n 2 pcs status
```

Confirm the cluster is healthy before starting — all green in Terminal 3, no errors in Terminals 1 and 2.

#### 5.2.2 Launch a test instance on compute2

```bash
# On the controller
openstack server create \
  --flavor m1.tiny \
  --image <your-image> \
  --availability-zone nova:compute2-vm \
  test-evacuation-vm

openstack server list --host compute2-vm
```

Confirm the instance is ACTIVE on compute2 before proceeding.

#### 5.2.3 Simulate compute2 failure

The most realistic test — network isolation, node stays powered on (simulating a hung/unresponsive node). Since compute2 is a virtual compute node, this can be simulated by disabling the security group governing its network access rather than using iptables:

```bash
# On the controller — disable/remove the security group rules allowing traffic to compute2
openstack security group rule list <compute2-security-group>
openstack security group rule delete <rule-id>
```

Removing the security group access cuts all connectivity from compute2 while leaving it technically powered on — the exact scenario where split-brain is a real risk without fencing.

Supplementary commands used during testing:

```bash
curl -s -X POST http://10.0.0.100:15868/v1/notifications \
  -H "X-Auth-Token: $(openstack token issue -c id -f value)" \
  -H "Content-Type: application/json" \
  -d '{"notification":{"type":"COMPUTE_HOST","hostname":"compute-2","generated_time":"'"$(date -u +%Y-%m-%dT%H:%M:%S)"'","payload":{"event":"STOPPED","host_status":"NORMAL","cluster_status":"OFFLINE"}}}'
```

```bash
sudo bash -c 'source /etc/kolla/admin-openrc.sh && openstack compute service set --down compute-1 nova-compute && openstack compute service list --host compute-1' 2>/dev/null
```

```bash
sudo bash -c 'source /etc/kolla/admin-openrc.sh && openstack server show test-evacuate-vm -c status -c "OS-EXT-SRV-ATTR:hypervisor_hostname" -c "OS-EXT-STS:vm_state" -c addresses -c "OS-EXT-STS:power_state" -f yaml' 2>/dev/null
```

#### 5.2.4 Watch the sequence in your terminals

The sequence you should observe (timing will vary based on Corosync timeouts):

**~30–60 seconds:** Terminal 3 shows compute2-vm dropping from Online to OFFLINE or UNCLEAN.

**~60–90 seconds:** Terminal 2 shows Pacemaker initiating STONITH:

```
notice:  Initiating fencing operation for node compute2-vm
notice:  Requesting fencing (off) of node compute2-vm
```

**~90 seconds:** `fence_ipmilan` fires against vBMC port 6232. Verify from the controller:

```bash
ipmitool -I lanplus -H 192.168.100.10 -p 6232 -U admin -P fence-secret power status
# Should now return: Chassis Power is off
```

**After fencing confirms:** Terminal 1 shows masakari-engine processing the host_failure and beginning evacuation:

```
host_failure notification received for compute2-vm
executing host_failure_workflow
```

**~2–5 minutes (depending on your evacuation timeout config):** The test instance should be ACTIVE on compute1:

```bash
openstack server list --host compute1-vm
```

---

## 6. Troubleshooting

| Symptom | Likely Cause / Check |
|---|---|
| compute2 shows UNCLEAN but fencing never fires | `fence_ipmilan` can't reach vBMC. Check `vbmc list` on the controller (still running?), check UDP port reachability from compute1 to controller port 6232. |
| Fencing fires but evacuation never starts | Masakari failover segment not configured, or masakari-hostmonitor isn't watching the right Corosync ring. Check `docker logs masakari_hostmonitor`. |
| Cluster loses quorum entirely (both nodes offline) | The delay parameter race. Check that `delay=15` is only on fence-compute1, not both. If both nodes fenced each other simultaneously, this is the cause. |
| `pcs stonith` test passes but real fencing fails | Check that the fence resource is actually running on the *other* node (location constraints working), and that the node trying to fence can reach vBMC. |

Additional reference — service status check during troubleshooting:

```bash
openstack compute service list --service nova-compute
+--------------------------------------+--------------+-----------+------+----------+-------+----------------------------+
| ID                                   | Binary       | Host      | Zone | Status   | State | Updated At                 |
+--------------------------------------+--------------+-----------+------+----------+-------+----------------------------+
| bb6f2c4f-02b9-49e1-bcb2-b0ec3998ac42 | nova-compute | compute-1 | nova | enabled  | up    | 2026-07-07T15:05:11.000000 |
| d1f7cf1e-fdc9-46da-b6db-4deb5e49bf69 | nova-compute | compute-2 | nova | disabled | up    | 2026-07-07T15:05:03.000000 |
+--------------------------------------+--------------+-----------+------+----------+-------+----------------------------+
```

Reference log sample — notification rejected because host is already under maintenance:

```
2026-07-09 16:32:17.147 384347 INFO masakarimonitors.ha.masakari [-] Send a notification. {'notification': {'type': 'COMPUTE_HOST', 'hostname': 'compute-2', 'generated_time': datetime.datetime(2026, 7, 9, 10, 47, 17, 147538), 'payload': {'event': 'STARTED', 'cluster_status': 'ONLINE', 'host_status': 'NORMAL'}}}
2026-07-09 16:32:17.257 384347 INFO masakarimonitors.ha.masakari [-] ConflictException: 409: Client Error for url: http://10.0.0.100:15868/v1/notifications, Notification received from host compute-2 of type 'COMPUTE_HOST' is ignored as the host is already under maintenance.
```

Reference log sample — engine disabling nova-compute before recovery:

```
2026-07-09 17:00:31.118 7 INFO masakari.compute.nova [req-9bb4f55f-ab95-4c48-b9bb-d353d7533f88 req-4792b022-753d-450b-9cef-9cee61dd1278 nova - - - - -] Disable nova-compute on compute-1
2026-07-09 17:00:31.252 7 INFO masakari.engine.drivers.taskflow.host_failure [req-9bb4f55f-ab95-4c48-b9bb-d353d7533f88 req-4792b022-753d-450b-9cef-9cee61dd1278 nova - - - - -] Sleeping 180 sec before starting recovery thread until nova recognizes the node down.
```

---

## 7. Recovery — Restoring a Fenced Compute Node

After a successful test, compute2 is powered off (libvirt VM is down). Bring it back:

```bash
# On the hypervisor host / controller
virsh start compute2-vm
```

Wait for the VM to boot, then on the now-running compute2:

```bash
# Remove the iptables block if it persisted across reboot (usually doesn't)
iptables -F

# Restart cluster services if they didn't auto-start
systemctl start corosync pacemaker
```

Verify compute2 rejoins the cluster:

```bash
# On compute1
pcs status
# compute2-vm should return to Online
```

Clear any stale Pacemaker failure state:

```bash
pcs resource cleanup
```

Confirm qdevice still shows 3 votes:

```bash
pcs quorum status
# Expected votes: 3, Total votes: 3
```

Decide whether to rebalance the evacuated instance back to compute2 — Masakari doesn't do this automatically. You can live-migrate it back manually if needed:

```bash
openstack server migrate --live-migration --host compute2-vm test-evacuation-vm
```

---

## 8. Appendix — Reference Log Samples

This section consolidates the raw log excerpts referenced throughout the document for quick lookup.

**Masakari monitor — maintenance conflict:**
```
2026-07-09 16:32:17.147 384347 INFO masakarimonitors.ha.masakari [-] Send a notification. {'notification': {'type': 'COMPUTE_HOST', 'hostname': 'compute-2', 'generated_time': datetime.datetime(2026, 7, 9, 10, 47, 17, 147538), 'payload': {'event': 'STARTED', 'cluster_status': 'ONLINE', 'host_status': 'NORMAL'}}}
2026-07-09 16:32:17.257 384347 INFO masakarimonitors.ha.masakari [-] ConflictException: 409: Client Error for url: http://10.0.0.100:15868/v1/notifications, Notification received from host compute-2 of type 'COMPUTE_HOST' is ignored as the host is already under maintenance.
```

**Masakari engine — service disable before recovery:**
```
2026-07-09 17:00:31.118 7 INFO masakari.compute.nova [req-9bb4f55f-ab95-4c48-b9bb-d353d7533f88 req-4792b022-753d-450b-9cef-9cee61dd1278 nova - - - - -] Disable nova-compute on compute-1
2026-07-09 17:00:31.252 7 INFO masakari.engine.drivers.taskflow.host_failure [req-9bb4f55f-ab95-4c48-b9bb-d353d7533f88 req-4792b022-753d-450b-9cef-9cee61dd1278 nova - - - - -] Sleeping 180 sec before starting recovery thread until nova recognizes the node down.
```