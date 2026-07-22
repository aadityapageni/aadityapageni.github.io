+++
title = "Get Octavia working in your Openstack"
date = 2026-07-22
[taxonomies]
tags = ["openstack","octavia"]

[extra]
mermaid = true
+++

I've been running my own OpenStack cluster on Kolla-Ansible for a while now, mostly because I like understanding the plumbing of things I depend on. Compute, block storage, networking, all of that I'd already fought with and made peace with. Load balancing was the one piece I kept avoiding. Everyone told me Octavia was "basically free" once Neutron was working. That turned out to be true in the same way that assembling furniture is "basically free" once you have all the parts. Sure. Except you don't know if you have all the parts until you're three hours in and missing a bolt.

This is the story of getting Octavia running with the Amphora provider on Kolla-Ansible, OpenStack 2025.1, OVN networking. It took me a weekend. It should have taken an afternoon. The gap between those two numbers is basically the whole post.

## What Octavia actually does?

Octavia doesn't load balance anything itself. What it does is spin up tiny VMs called Amphorae, each one just running HAProxy, and then it manages those VMs over a private network you probably didn't know you needed until right now.

Every Amphora has two legs:

- a management port, on `lb-mgmt-net`, used for health checks, pushing HAProxy config, and cert rotation
- a VIP port, on your actual tenant network, which is where client traffic comes in and backend traffic goes out

In my setup that looked like this:

| Network | Subnet | Type | Purpose |
|---|---|---|---|
| `lb-mgmt-net` | 10.1.0.0/24 | VLAN 100 (physnet1) | Octavia control plane only |
| `LAN1` | 10.0.0.0/24 | tenant overlay | Backend VMs + LB VIP |
| `public1` | 10.20.30.0/24 | provider flat | External/floating IPs |

And the IP allocation ended up looking like this, once everything was working:

| Component | IP | Why |
|---|---|---|
| `o-hm0` (controller) | 10.1.0.51 | Health-manager endpoint, manually assigned, outside the DHCP pool |
| Amphora mgmt port | 10.1.0.100–200 (DHCP) | Auto-assigned from the `lb-mgmt-subnet` pool |
| Amphora vip port | e.g. 10.0.0.64 | Auto-assigned from `LAN1`, HAProxy uses this to reach backends |
| Load Balancer VIP | e.g. 10.0.0.77 | Auto-assigned from `LAN1` when the LB is created |

The part that isn't obvious from any of that is the interface named `o-hm0`. It's a single OVS internal port on the controller that gives Octavia's health-manager container a physical presence on `lb-mgmt-net`. Without it, the health-manager can bind its socket, listen on its port, log "healthy," and still be completely unreachable from the network it thinks it's on. Nobody tells you this up front. You find out.

Here's the path traffic takes once it's actually wired up correctly:

```
octavia_health_manager container
        │
        │ binds to 10.1.0.51:5555
        ▼
    o-hm0  (OVS internal port on br-ex, tag=100)
        │
        │ VLAN 100 tagged frames exit via ens19
        ▼
    br-ex → ens19 → physical switch
        │
        ▼
    Amphora VM mgmt port (10.1.0.x on lb-mgmt-net VLAN 100)
        │ port 9443 — HAProxy agent API
        │ port 5555 UDP — heartbeat back to 10.1.0.51
```

I stared at this diagram (well, my own scribbled version of it) for a solid twenty minutes at one point trying to figure out which hop was broken. Turns out the answer was "the second one," but I'm getting ahead of myself.

## Getting the config in place

The globals.yml changes are the easy part. This is what I ended up with:

```yaml
####################
# Neutron (required for VLAN lb-mgmt-net)
####################
neutron_physical_networks: "physnet1"
neutron_ovn_bridge_mappings: "physnet1:br-ex"
neutron_ovn_vlan_ranges: "physnet1:1:4094"

####################
# Octavia
####################
enable_octavia: "yes"

# Provider driver — must be NAME:Description format
octavia_provider_drivers: "amphora:Amphora provider"

# Host interface the health-manager binds to
octavia_network_interface: "o-hm0"

# Ensure o-hm0 is tagged with the lb-mgmt VLAN on OVS
octavia_hm_ovs_vlan: 100

# Let kolla create lb-mgmt-net, flavor, and security groups automatically
octavia_auto_configure: "yes"

# Must match the --tag used when uploading the Amphora image to Glance
octavia_amp_image_tag: "amphora"

# Amphora VM flavor — increase RAM to 1024 to avoid boot failures
octavia_amp_flavor:
  name: "amphora"
  is_public: no
  vcpus: 1
  ram: 1024
  disk: 5

# lb-mgmt-net definition — kolla creates this automatically when octavia_auto_configure: yes
# NOTE: use provider_segmentation_id, not provider_segment
octavia_amp_network:
  name: "lb-mgmt-net"
  provider_network_type: "vlan"
  provider_physical_network: "physnet1"
  provider_segmentation_id: 100
  external: false
  shared: false
  subnet:
    name: "lb-mgmt-subnet"
    cidr: "10.1.0.0/24"
    allocation_pool_start: "10.1.0.100"
    allocation_pool_end: "10.1.0.200"
    enable_dhcp: yes

####################
# Horizon
####################
horizon_enable_octavia_ui: "yes"
```

Small note that cost me a search through the source once: it's `provider_segmentation_id`, not `provider_segment`. Easy to typo, and if you get it wrong the network just won't get the VLAN tag you expect, with no error telling you why.

If you're doing this on more than one controller, every node needs the identical globals.yml, and every node needs its own `o-hm0` setup done individually, with its own static IP outside the DHCP range (10.1.0.51 on the first controller, 10.1.0.52 on the second, and so on). More on that later, because multinode is where this whole thing gets genuinely annoying.

## Pre-deployment: certs and the Amphora image

Octavia does mutual TLS between the health-manager and every Amphora VM, which is a nice touch and also one more thing that can be subtly wrong. Generating the certs is one command:

```bash
kolla-ansible octavia-certificates
# Certificates are written to /etc/kolla/config/octavia/
ls /etc/kolla/config/octavia/
# Expected: client_ca.cert.pem  client.cert-and-key.pem
#           server_ca.cert.pem  server_ca.key.pem
```

Then you need an actual Amphora image, because Octavia doesn't ship one. You build it yourself with diskimage-builder:

```bash
pip install diskimage-builder --break-system-packages

git clone https://github.com/openstack/octavia.git
cd octavia/diskimage-create/
bash diskimage-create.sh
```

This takes somewhere between five and ten minutes depending on your machine, and produces a qcow2 image you then upload to Glance:

```bash
source /etc/kolla/admin-openrc.sh
openstack image create amphora-x64-haproxy \
  --public \
  --container-format bare \
  --disk-format qcow2 \
  --file /path/to/amphora-x64-haproxy.qcow2 \
  --tag amphora
```

The `--public` flag matters more than it looks like it should. Without it, Octavia's service account can end up unable to find the image at all, because it's not looking in your project, it's looking as a service user. I didn't hit this myself, but it's exactly the kind of thing that would have cost me another hour if I had, so I made the image public from the start and moved on.

If your Neutron ML2 setup doesn't already have VLAN ranges configured (mine mostly did, but I checked anyway):

```bash
mkdir -p /etc/kolla/config/neutron/
cat > /etc/kolla/config/neutron/ml2_conf.ini << 'EOF'
[ml2_type_vlan]
network_vlan_ranges = physnet1:1:4094
EOF
```

And I added this to the Octavia config directly, mostly out of an abundance of caution, before the actual bug revealed itself:

```bash
cat >> /etc/kolla/config/octavia/octavia.conf << 'EOF'

[health_manager]
bind_ip = 10.1.0.51
controller_ip_port_list = 10.1.0.51:5555
EOF
```

## The o-hm0 setup, which is the part nobody warns you about

This is the section of the whole process that actually matters, and it's also the part that's easiest to skip because nothing in the deploy step will complain if you skip it. Kolla's `octavia_auto_configure: yes` creates the Neutron network, the subnet, the security groups, the flavor. It does not create the host-level OVS port that lets your health-manager container actually reach any of that. You have to do that by hand.

First, anchor the health-manager's MAC address in Neutron, so OVN programs the right flows for it:

```bash
source /etc/kolla/admin-openrc.sh

openstack port create \
  --network lb-mgmt-net \
  --device-owner Octavia:health-mgr \
  --fixed-ip ip-address=10.1.0.51 \
  --no-security-group \
  octavia-health-manager-port-$(hostname)

HM_MAC=$(openstack port show octavia-health-manager-port-$(hostname) \
  -f value -c mac_address)
echo "MAC: $HM_MAC"
```

Then create the actual OVS port and give it that MAC:

```bash
ovs-vsctl --if-exists del-port o-hm0

ovs-vsctl add-port br-ex o-hm0 tag=100 -- set Interface o-hm0 type=internal

ip link set o-hm0 down
ip link set o-hm0 address "$HM_MAC"
ip link set o-hm0 up

ip addr add 10.1.0.51/24 dev o-hm0 2>/dev/null || true

ovs-vsctl port-to-br o-hm0          # Must print: br-ex
ovs-vsctl list port o-hm0 | grep tag # Must show:  tag: 100
```

The key rule, which I'm putting in its own paragraph because I violated it and paid for it: `o-hm0` has to live on `br-ex`, tagged with VLAN 100, and OVS handles that tagging internally. Do not, under any circumstances, also create a host-level VLAN subinterface for the same tag, like `ens18.100` via netplan. I'll get to why in a minute. It is not a hypothetical warning.

Since none of this survives a reboot on its own, I wrapped it in a systemd unit:

```bash
cat > /etc/systemd/system/o-hm0.service << EOF
[Unit]
Description=Octavia health-manager OVS port (o-hm0)
After=network.target openvswitch-switch.service
Requires=openvswitch-switch.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c ' \\
  ovs-vsctl --if-exists del-port o-hm0; \\
  ovs-vsctl --may-exist add-port br-ex o-hm0 tag=100 -- set Interface o-hm0 type=internal; \\
  ip link set o-hm0 address ${HM_MAC}; \\
  ip addr replace 10.1.0.51/24 dev o-hm0; \\
  ip link set o-hm0 up'
ExecStop=/bin/bash -c 'ovs-vsctl --if-exists del-port br-ex o-hm0'

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now o-hm0.service
systemctl status o-hm0.service
```

One thing that'll bite you if you're copy-pasting this from notes like I was: replace `${HM_MAC}` in the file with the literal MAC address you got back earlier. Systemd unit files don't interpolate shell variables from your current session, obviously, but it's the kind of mistake you make at 11pm when you've written the same MAC address six times already and your fingers are on autopilot.

## Actually deploying the thing

```bash
# If deploying Octavia for the first time on an existing cluster:
kolla-ansible reconfigure -t octavia,horizon

# If this is a fresh first deploy:
kolla-ansible deploy -i all-in-one --tags octavia
kolla-ansible post-deploy -i all-in-one

docker ps --format "table {{.Names}}\t{{.Status}}" | grep octavia
```

And you're looking for five containers, all healthy:

```
octavia_api              Up X minutes (healthy)
octavia_worker           Up X minutes (healthy)
octavia_health_manager   Up X minutes (healthy)
octavia_housekeeping     Up X minutes (healthy)
octavia_driver_agent     Up X minutes (healthy)
```

Mine came up exactly like that. All green. Which felt like the finish line. It was not the finish line.

## Post-deploy checks I actually ran

Before touching anything load-balancer shaped, I went through the basic validation:

```bash
docker exec octavia_health_manager grep bind_ip /etc/octavia/octavia.conf
# Expected: bind_ip = 10.1.0.51

docker exec octavia_health_manager ss -ulnp | grep 5555
# Expected: udp UNCONN 0 0 10.1.0.51:5555

ovs-vsctl port-to-br o-hm0       # br-ex
ovs-vsctl list port o-hm0 | grep tag  # tag: 100
```

Then the provider and resource checks:

```bash
source /etc/kolla/admin-openrc.sh

openstack loadbalancer provider list
# Expected: amphora | Amphora provider

openstack network show lb-mgmt-net | grep -E "provider:|id"
# Expected: provider:network_type = vlan, provider:segmentation_id = 100

openstack security group list | grep octavia
openstack flavor list --all | grep amphora
```

And a look at the worker logs, which should end on something reassuring:

```bash
docker logs octavia_worker 2>&1 | tail -20
# Should end with: "Starting V2 consumer..."
# Must NOT contain: ConfigFileValueError or CRITICAL
```

Every single one of these came back clean. Provider registered, network tagged correctly, flavor present, worker logs boring in the good way. I remember genuinely thinking, at this point, that I was basically done.

## Creating an actual load balancer, and then waiting

```bash
source /etc/kolla/admin-openrc.sh

SUBNET_ID=$(openstack subnet show <your-tenant-subnet-name> -f value -c id)

openstack loadbalancer create \
  --name web-lb \
  --vip-subnet-id $SUBNET_ID \
  --wait

openstack loadbalancer listener create \
  --name web-listener \
  --protocol HTTP \
  --protocol-port 80 \
  --wait \
  web-lb

openstack loadbalancer pool create \
  --name web-pool \
  --lb-algorithm ROUND_ROBIN \
  --listener web-listener \
  --protocol HTTP \
  --wait

openstack loadbalancer member create \
  --name lb-1-1 \
  --subnet-id $SUBNET_ID \
  --address 10.0.0.44 \
  --protocol-port 80 \
  web-pool

openstack loadbalancer member create \
  --name lb-1-2 \
  --subnet-id $SUBNET_ID \
  --address 10.0.0.59 \
  --protocol-port 80 \
  web-pool

openstack loadbalancer healthmonitor create \
  --name web-health \
  --delay 5 \
  --max-retries 4 \
  --timeout 10 \
  --type HTTP \
  --url-path / \
  web-pool

watch openstack loadbalancer show web-lb
```

At this point it's worth knowing what you're actually watching for, because the statuses are not self-explanatory the first time you see them:

| Status Field | Value | Meaning |
|---|---|---|
| Provisioning Status | `ACTIVE` | Infrastructure built successfully, Amphora booted, ports wired |
| Provisioning Status | `PENDING_CREATE` | Amphora VM is still booting, wait 2–3 min |
| Operating Status | `ONLINE` | Health monitor is passing, LB is routing traffic |
| Operating Status | `OFFLINE` | No health monitor attached, or health checks failing |
| Operating Status | `ERROR` | Backend unreachable, check security groups |
| Member Status | `NO_MONITOR` | Health monitor not yet created |

`PENDING_CREATE` is supposed to last two or three minutes. That's the Amphora booting, nothing more. Mine sat there for twenty. Then thirty. I made a coffee. It was still `PENDING_CREATE` when I got back.

## Where it actually broke

Nothing crashed. That's what made this annoying. Every container was healthy. Nova said the Amphora VM was running fine. If I hadn't been specifically watching the load balancer status, I would have had no reason to believe anything was wrong at all.

```bash
docker logs octavia_worker 2>&1 | grep -E "ERROR|WARNING" | tail -30
```

Nothing useful. A few warnings, none of them related.

```bash
openstack server list --all-projects | grep amphora
```

The VM existed. Booted. Had an IP from the DHCP pool on `lb-mgmt-net`. From Nova's side of the world, this was a completely unremarkable, successful VM boot.

```bash
openstack port list --network lb-mgmt-net
```

Also fine. The port existed, had an address in the expected range.

Then the one that actually told me something:

```bash
docker logs octavia_health_manager 2>&1 | grep -i heartbeat
```

Nothing. Not "heartbeat failed," not "heartbeat timeout." Just no lines at all, ever, matching that word. The Amphora was out there, alive, presumably trying to phone home, and the health-manager had simply never heard from it. Not even once.

The common causes for exactly this symptom, according to the troubleshooting notes I'd already collected: Amphora image missing or tagged wrong, wrong `amp_boot_network_list`, or `o-hm0`'s VLAN tag missing. I'd built the image correctly and tagged it. So I went and checked the VLAN tag.

```bash
ovs-vsctl list port o-hm0 | grep tag
```

Empty. No tag at all.

## The wrong turn

My first working theory, once I saw the missing tag, was reasonable enough: the OVS port itself must be misconfigured somehow, maybe the internal type wasn't sticking, maybe I needed a more explicit interface on the host side to actually carry VLAN 100. So I did something I now recognize as a mistake almost the instant I typed it: I added a Netplan VLAN subinterface on top of the physical NIC, `ens18.100`, thinking OVS and a kernel-level VLAN interface could just coexist.

They cannot. Within about five minutes `br-ex` stopped responding to anything at all, because now there were two separate things trying to own VLAN 100 tagging on the same physical link, one via OVS's internal tagging table and one via a kernel VLAN device sitting on top of the same NIC. I reverted the netplan change as fast as I could type the commands, and connectivity came back once I did.

That detour cost me more time than the actual bug did. Worth saying plainly: when something is OVS-managed, the fix is basically never "also manage it at the kernel level in parallel." Pick one owner for VLAN tagging. OVS was already supposed to be that owner. I just hadn't actually told it to be.

## The real fix

Once I stopped trying to be clever, the fix was almost embarrassingly small. Set the tag directly on the OVS port object, nothing else:

```bash
ovs-vsctl set port o-hm0 tag=100
ip link set o-hm0 up
```

And then test actual reachability, not just config state:

```bash
AMPHORA_IP=$(openstack port list --network lb-mgmt-net -f value -c fixed_ips \
  | grep -oP '10\.1\.0\.\d+' | head -1)
ping -c3 -I o-hm0 $AMPHORA_IP
```

The ping came back. First time all day something on this network path actually answered. I checked the heartbeat log again more out of habit than expectation, and this time there were lines. The load balancer moved to `ACTIVE` within about thirty seconds of that.

## Other things that go wrong, that I ran into or read about while debugging

A few other failure modes showed up along the way, or were close enough calls that I want to mention them.

Members going `OFFLINE` or `ERROR` right after you attach a health monitor is usually not the health monitor's fault. The Amphora probes your backends from its VIP port on the tenant network, and if your backend VMs' security groups don't allow inbound traffic from that Amphora, the health checks just fail silently and the member sits there looking broken when the backend itself is fine.

```bash
openstack server show lb-1-1 -c security_groups

openstack security group rule create \
  --protocol tcp \
  --dst-port 80 \
  --remote-ip 10.0.0.0/24 \
  <security-group-id>
```

I also hit the classic typo in `octavia_provider_drivers`. If the worker logs show:

```
ConfigFileValueError: Value should be NAME:VALUE pairs separated by ","
```

it's because the value needs to be `NAME:Description`, not just the bare name:

```yaml
# WRONG:
octavia_provider_drivers: "amphora"

# CORRECT:
octavia_provider_drivers: "amphora:Amphora provider"
```

Then `kolla-ansible reconfigure -t octavia` and it clears up.

And if you ever need to change `lb-mgmt-net`'s provider network type after the fact, you'll hit this:

```
The following parameters cannot be updated: provider_network_type
```

Neutron just won't let you update that field in place. The only real fix is to delete the network entirely and let Kolla recreate it:

```bash
source /etc/kolla/admin-openrc.sh
NET_ID=$(openstack network show lb-mgmt-net -f value -c id)
openstack port list --network $NET_ID -f value -c id | xargs -r openstack port delete
openstack subnet delete lb-mgmt-subnet
openstack network delete lb-mgmt-net
kolla-ansible reconfigure -t octavia
```

## Multinode, which is its own separate headache

Everything above was single-controller. On a multinode cluster, the health-manager runs on every controller, and all of them form an active-active cluster where any of them can receive a heartbeat from any Amphora. Which sounds nice until you realize it means you have to repeat the entire `o-hm0` dance on every single node, individually, with a unique IP each time.

| Node | o-hm0 IP | Neutron Port Name |
|---|---|---|
| controller-1 | 10.1.0.51 | octavia-health-manager-port-controller-1 |
| controller-2 | 10.1.0.52 | octavia-health-manager-port-controller-2 |
| controller-3 | 10.1.0.53 | octavia-health-manager-port-controller-3 |

All of those IPs need to sit outside the DHCP allocation range (100 through 200), or you'll get conflicts with whatever the Amphorae are handed automatically.

You also need to tell Octavia about every controller's health-manager endpoint, via an override:

```ini
[health_manager]
controller_ip_port_list = 10.1.0.51:5555,10.1.0.52:5555,10.1.0.53:5555
```

saved to `/etc/kolla/config/octavia.conf`, followed by `kolla-ansible reconfigure -t octavia`.

Kolla already sets `bind_ip` per host using `{{ api_interface_address }}`, which resolves correctly to each node's own management IP without you needing to touch it. What you do need to get right is `controller_ip_port_list` listing all the nodes, because that's how each Amphora knows every place it's allowed to send its heartbeat.

The Amphora image itself needs to stay `--public` (or at least shared to the `service` project), since any controller's worker might be the one spawning a given Amphora, and it needs to be able to find the image regardless of which node that ends up being.

And finally, every controller's `o-hm0` needs to actually be verified individually, not just assumed to be fine because node one worked:

```bash
# Run on each controller:
ovs-vsctl port-to-br o-hm0        # must be br-ex
ovs-vsctl list port o-hm0 | grep tag  # must be tag: 100
ping -c3 -I o-hm0 10.1.0.1        # basic L2 reachability test
```

If even one node can't reach `lb-mgmt-net`, any Amphora that happens to get assigned to that node will never send it a heartbeat, and you'll see false `OFFLINE` statuses that have nothing to do with the actual backend health. I didn't run a three-node cluster myself for this particular exercise, but I made sure to write all of this down precisely because I know I'll be doing it eventually, and I will not remember any of it by then.

## What actually surprised me, thinking back on it

The VLAN tag itself isn't the interesting part. I'll forget the exact detail within a year, honestly, and have to relearn it from these exact notes. What stuck with me is how quiet the whole failure was, at every single layer.

Nova didn't complain, the VM booted fine. Neutron didn't complain, the ports and networks all looked correct in every single `show` and `list` command I ran. Octavia's own containers reported healthy the entire time. The only actual signal that anything was wrong was the absence of one specific log line, a line that has no corresponding warning explaining why it's missing. There's no "heartbeat expected, not received" message. There's just silence, which I suppose is technically accurate, since silence is literally what was happening on the wire.

That's a deliberate design choice and not really a bug, since health-manager is built to tolerate Amphorae that take a while to boot, so it doesn't scream the second a heartbeat is a little late. But it does mean you get almost no signal to work backward from. You more or less have to already carry the entire traffic path around in your head, health-manager to o-hm0 to br-ex to the VLAN tag to the physical switch to the Amphora's mgmt port, and check every hop manually, because nothing is going to point you at the broken one.

## Numbers, for what they're worth

This wasn't really a performance exercise, the bottleneck the whole time was configuration, not throughput, but a few things worth noting anyway:

Amphora boot to `ACTIVE` took roughly 2 to 3 minutes once the network path actually existed, and that's basically all VM boot time, nothing Octavia-specific about it.

The health monitor settings I landed on: 5 second delay, 4 max retries, 10 second timeout. Aggressive enough to catch a dead backend inside about 20 seconds without flapping on some momentary blip.

Once the path existed, heartbeats came in every few seconds, steady. The failure mode was never "slow." It was "never," which in hindsight is exactly why staring at timing graphs wouldn't have helped me at all.

A single Amphora on a 1 vCPU, 1024 MB flavor is fine for the kind of traffic I actually care about. I wouldn't trust it for anything serious without bumping that flavor up first, but that's a separate problem from the one this post is about.

## What I'd do differently

I'd write the `o-hm0` checks as a small script and run them before deploying Octavia at all, not after something's already stuck in `PENDING_CREATE` for half an hour:

```bash
ovs-vsctl port-to-br o-hm0          # expect: br-ex
ovs-vsctl list port o-hm0 | grep tag # expect: tag: 100
ip addr show o-hm0                  # expect: 10.1.0.51/24, UP
```

I would not touch Netplan. At all. That detour cost more time than the real bug, and the lesson generalizes further than just this one interface: if OVS owns something, don't also try to manage it from the kernel side in parallel, even "just to check."

And on multinode, I'd set up every controller's `o-hm0` and its `controller_ip_port_list` entry in the same sitting, rather than getting node one working and assuming the rest would just follow along. They don't. Each one needs its own Neutron port, its own IP outside the DHCP pool, and its own OVS setup, or you'll end up with Amphorae quietly heartbeating into nothing on whichever node you forgot.

## Closing

I don't think Octavia is badly built. The Amphora-plus-health-manager model is a reasonable way to get a real HAProxy instance per load balancer without inventing a custom control plane from scratch. But the amount of manual setup it expects from whoever's running Kolla-Ansible is bigger than the quick-start docs make it sound, and almost every failure mode I ran into was silent rather than loud.

I still don't know if the OVS-internal-port approach is really the right long-term answer compared to a proper Neutron trunk port, and I haven't tried the OVN-native provider driver seriously enough yet to know whether its L7 gaps have closed since I last looked. If you've run Octavia in production on Kolla-Ansible and ended up with a different pattern for the management network, I'd genuinely like to hear what broke for you, and how you figured out it was broken in the first place.