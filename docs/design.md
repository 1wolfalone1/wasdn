# Design Overview

## The Lore

Every module is named after something from Elden Ring / Dark Souls:

| Module | What it does | The Reference |
|---|---|---|
| **Roundtable** | Control plane | Roundtable Hold — where decisions are made |
| **Minoreardtree** | Node agent | Minor Erdtrees — one in each region, connected to the Great Tree |
| **Bonfire** | L2 switch | Bonfires — connection points |
| **Fogwall** | Firewall / ACL | Fog walls — you shall not pass |
| **Deeproot** | VXLAN overlay tunnels | Deeproot Depths — underground tunnels connecting everything |
| **Waygate** | L3 routing, NAT, floating IPs | Waygates — portals between distant regions |
| **Flask** | DHCP server for VMs | Estus Flask / Flask of Crimson Tears — gives life |
| **Ember** | Packet parser | Embers — reveal what's hidden |
| **Grace** | Network device I/O | Sites of Grace — entry points |
| **Kindling** | Common utilities | The foundation that starts the first flame |
| **Ashofwar** | eBPF/XDP kernel-space data plane | Ashes of War — changes how your weapon works |

## Architecture

### High Level

```
ROUNDTABLE (control plane, runs on master)
    |
    |  commands: create network/subnet/router/port, apply rules
    |
    v
MINOREARDTREE --------- MINOREARDTREE --------- MINOREARDTREE
(Node 1 agent)          (Node 2 agent)          (Node N agent)
    |                       |                       |
    |  <--- DEEPROOT (VXLAN tunnels between nodes) --->
    |                       |                       |
 [WAYGATE]               [WAYGATE]               [WAYGATE]
 [BONFIRE]               [BONFIRE]               [BONFIRE]
 [FOGWALL]               [FOGWALL]               [FOGWALL]
 [FLASK]                 [FLASK]                 [FLASK]
 [GRACE]                 [GRACE]                 [GRACE]
    |                       |                       |
 VM1  VM2                VM3  VM4                VM5  VM6
```

### Inside Each Node

```
MINOREARDTREE (agent daemon)
  |
  |-- WAYGATE --- L3 routing, NAT, floating IPs
  |
  |-- BONFIRE --- L2 switch, MAC learning, forwarding
  |
  |-- FOGWALL --- packet filtering, ACL, security groups
  |
  |-- DEEPROOT -- VXLAN encap/decap for cross-node traffic
  |
  |-- FLASK ----- DHCP server, assigns IPs to VMs
  |
  |-- GRACE ----- reads/writes raw frames from tap/veth
  |
  |-- EMBER ----- parses packet bytes into structured headers
  |
  |-- KINDLING -- logging, config, shared utils
```

### Where Everything Runs

```
wasoulcloud cluster
|
|-- K8s master node
|     |-- roundtable-server (control plane API)
|
|-- K8s worker node 1
|     |-- minoreardtree (agent)
|     |-- KubeVirt VMs
|
|-- K8s worker node 2
|     |-- minoreardtree (agent)
|     |-- KubeVirt VMs
|
|-- Ceph nodes (storage, separate from greattree)
```

## Packet Flow

### Same Subnet (L2) — VM-A and VM-B on 10.0.1.0/24

```
VM-A sends packet
  |
  v
GRACE --- reads raw frame from tap device
  |
  v
EMBER --- parses: dst=VM-B, src=VM-A
  |
  v
FOGWALL - outbound check: allowed? --> yes
  |
  v
BONFIRE - MAC lookup: VM-B not on this node
  |
  v
DEEPROOT wraps frame in VXLAN, sends UDP to Node 2
  |
  ====== physical network ======
  |
  v
DEEPROOT unwraps VXLAN on Node 2
  |
  v
BONFIRE - MAC lookup: VM-B on local port 2
  |
  v
FOGWALL - inbound check: allowed? --> yes
  |
  v
GRACE --- writes frame to VM-B's tap device
  |
  v
VM-B receives packet
```

### Cross Subnet (L3) — VM-A on 10.0.1.0/24, VM-B on 10.0.2.0/24

```
VM-A (10.0.1.10) sends packet to VM-B (10.0.2.20)
  |
  v
GRACE --- reads raw frame
  |
  v
EMBER --- parses: dst MAC is the virtual router (waygate)
  |
  v
FOGWALL - outbound check: allowed? --> yes
  |
  v
BONFIRE - MAC lookup: dst is router, send to WAYGATE
  |
  v
WAYGATE - route lookup: 10.0.2.0/24 exists
        - rewrites src/dst MAC for next hop
        - if floating IP: applies NAT
  |
  v
BONFIRE - forward to correct port (local or tunnel)
  |
  v
DEEPROOT (if VM-B is on another node)
  |
  ====== physical network ======
  |
  v
VM-B receives packet
```

### VM Boot (DHCP) — VM-A starts up

```
VM-A sends DHCP discover (broadcast)
  |
  v
GRACE --- reads broadcast frame
  |
  v
EMBER --- parses: this is a DHCP request
  |
  v
FLASK --- looks up subnet pool, assigns 10.0.1.10
        - responds with IP, gateway, DNS
  |
  v
VM-A now has an IP address
```

## Dependency Graph

```
       roundtable
           |
           v
      minoreardtree
      |  |  |  |  |
      v  v  v  v  v
waygate bonfire fogwall deeproot flask
      |    |    |    |     |
      v    v    v    v     v
         ember    grace
            |      |
            v      v
           kindling

ashofwar (separate path)
  provides alternative implementations:
  - EbpfTable    implements IForwardingTable (replaces bonfire internals)
  - EbpfFilter   implements IFirewall (replaces fogwall internals)
```

## Project Structure

```
greattree/
├── libs/
│   ├── kindling/       # Common utilities
│   ├── ember/          # Packet parsing
│   ├── grace/          # Network device I/O
│   ├── bonfire/        # L2 switch engine
│   ├── fogwall/        # Firewall / ACL
│   ├── deeproot/       # VXLAN overlay tunnels
│   ├── waygate/        # L3 routing, NAT, floating IPs
│   ├── flask/          # DHCP server
│   ├── ashofwar/       # eBPF/XDP alternative implementations
│   └── roundtable/     # Control plane logic
├── apps/
│   ├── minoreardtree/      # Node agent daemon
│   ├── roundtable-server/  # Control plane API server
│   └── gt-cli/             # Debug CLI
└── tests/
    ├── unit/
    └── integration/
```
