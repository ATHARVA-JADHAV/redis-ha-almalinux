# 03 - Redis Cluster Architecture and Preparation

## Objective

Plan the six Redis server processes that will form a Redis Cluster with three primaries and three replicas. This document explains the topology, network ports, and Redis Cluster concepts before any cluster configuration is applied.

The cluster has not yet been created. The port layout below is the intended design and will be verified during implementation.

## Planned VM and node layout

Each VM will run two independent Redis server processes. Each process is a separate Redis node with its own configuration, data directory, log, listening port, and cluster node configuration file.

| VM | Host-only address | Planned Redis node | Intended role |
|---|---|---:|---|
| redis-vm1 | 192.168.56.101 | 7000 | Primary 1 |
| redis-vm1 | 192.168.56.101 | 7001 | Replica |
| redis-vm2 | 192.168.56.102 | 7002 | Primary 2 |
| redis-vm2 | 192.168.56.102 | 7003 | Replica |
| redis-vm3 | 192.168.56.103 | 7004 | Primary 3 |
| redis-vm3 | 192.168.56.103 | 7005 | Replica |

The intended placement is for each replica to protect a primary running on a different VM. The actual assignments must be checked using Redis Cluster node information after cluster creation; the command-line cluster creator chooses replica assignments, so the final mapping must not be assumed from port order alone.

## Why six processes on three VMs?

A Redis Cluster distributes data across primary nodes. Replicas maintain copies of primary data and can be promoted if a primary fails and the cluster's failure-detection and voting requirements are met.

Running two Redis processes per VM lets this lab exercise a six-node topology using three virtual machines. This is a learning topology, not equivalent to six independent physical failure domains: if one VM fails, both Redis processes on that VM are lost.

## Client port and Cluster bus port

Each Redis Cluster node uses two TCP ports for different purposes.

### 1. Client/data port

The client port accepts Redis commands from applications and tools such as `redis-cli`. In this lab, the planned client ports are:

- 7000
- 7001
- 7002
- 7003
- 7004
- 7005

A client connects to a node using its Host-only IP address and client port.

### 2. Cluster bus port

The Cluster bus is a separate TCP communication channel used by Redis nodes to communicate with other nodes in the same cluster. It uses Redis's internal binary protocol; it is not a port for application clients or ordinary Redis commands.

Nodes use the bus to exchange cluster information, send gossip messages, check node liveness, detect failures, propagate configuration changes, and coordinate failover-related operations.

By default, Redis derives the Cluster bus port by adding 10,000 to the node's client port. Therefore, the planned bus ports are:

| Client port | Default Cluster bus port |
|---:|---:|
| 7000 | 17000 |
| 7001 | 17001 |
| 7002 | 17002 |
| 7003 | 17003 |
| 7004 | 17004 |
| 7005 | 17005 |

The `cluster-port` setting can override the default bus port, but this lab will use the default derivation unless implementation testing gives us a reason to change it.

**Important:** Both the client port and its corresponding Cluster bus port must be reachable between the Redis nodes. Opening only the client ports is not sufficient for a functioning cluster. The bus ports should be reachable from the other Redis nodes on the private Host-only network, not exposed to the public internet.

## What is a full mesh?

Redis Cluster nodes maintain node-to-node connections over the Cluster bus. In a six-node cluster, every node must be able to communicate with the other five nodes.

This connected full-mesh arrangement allows nodes to exchange cluster state and detect when another node becomes unavailable.

The Cluster bus is therefore different from the client/data port:

- Client/data port: application and Redis command traffic.
- Cluster bus port: internal Redis node-to-node coordination.

## Hash slots and sharding

Redis Cluster divides its keyspace into exactly 16,384 hash slots, numbered 0 through 16,383.

For a normal key, Redis calculates a CRC16 hash and takes the result modulo 16,384:

```text
HASH_SLOT = CRC16(key) mod 16384
```

Each primary owns a portion of these slots. When a client writes a key, the key's slot determines which primary is responsible for it. This is called sharding: the dataset is distributed across multiple primary nodes rather than stored entirely on one server.

A healthy, fully covered cluster should report all 16,384 slots assigned. Redis Cluster clients can follow redirections such as `MOVED` when a request reaches a node that does not own the requested key's slot.

## Replication and failover

Each replica follows one primary and maintains a copy of its dataset. Redis replication is asynchronous, so a primary failure can result in a small window of acknowledged writes not being present on the promoted replica.

Redis Cluster uses node failure detection and replica promotion to provide availability, but it is not a guarantee of zero data loss. We will deliberately stop a primary later and observe the failover behavior.

## Planned firewall access

The Redis nodes will communicate over the private Host-only network. The intended TCP port ranges are:

| Port range | Purpose | Expected source |
|---|---|---|
| 7000-7005 | Redis client/data connections | Lab clients and Redis nodes as required |
| 17000-17005 | Redis Cluster bus | Other Redis nodes in the lab |

We will inspect the active firewalld zone and interface assignment before adding rules. Rules will be scoped to the private lab network wherever practical. We will not expose Redis ports through the VirtualBox NAT adapter or to the public internet.

## Next implementation checks

Before creating instance configuration files, inspect the package-provided layout on each VM:

```bash
rpm -ql redis
systemctl list-unit-files | grep -i redis
systemctl status redis
```

Then verify that the installed Redis server and CLI versions match across the three VMs:

```bash
redis-server --version
redis-cli --version
```

The exact package paths, service unit, and version outputs will guide the instance configuration. We will not overwrite the packaged default configuration blindly.

## Cluster creation verification criteria

The cluster stage will be considered complete only after the following checks succeed:

- Six Redis nodes are reachable.
- Three nodes have the primary role and three have the replica role.
- Each replica follows a primary on a different VM, where the topology permits.
- All 16,384 hash slots are assigned.
- `cluster_state:ok` is reported.
- The node list shows the expected addresses, ports, roles, and connected links.

**Next:** Inspect the installed Redis package and service layout, then create and validate the six instance configurations.
