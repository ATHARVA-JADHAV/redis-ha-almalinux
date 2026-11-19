# 03 - Redis Cluster Architecture and Implementation

## Objective

Deploy and understand a six-node Redis Cluster across three AlmaLinux 9.8 virtual machines. The cluster contains three primaries and three replicas, with each replica placed on a different VM from its primary.

## Lab topology

Each VM runs two independent Redis server processes. Every process has its own configuration file, data directory, log file, client port, and Cluster node configuration file.

| VM | Host-only address | Redis port | Role |
|---|---|---:|---|
| redis-vm1 | 192.168.56.101 | 7000 | Primary |
| redis-vm1 | 192.168.56.101 | 7001 | Replica of VM3:7004 |
| redis-vm2 | 192.168.56.102 | 7002 | Primary |
| redis-vm2 | 192.168.56.102 | 7003 | Replica of VM1:7000 |
| redis-vm3 | 192.168.56.103 | 7004 | Primary |
| redis-vm3 | 192.168.56.103 | 7005 | Replica of VM2:7002 |

### Replica assignments

The following assignments were reported by the cluster creation and node inspection:

| Primary | Replica |
|---|---|
| 192.168.56.101:7000 | 192.168.56.102:7003 |
| 192.168.56.102:7002 | 192.168.56.103:7005 |
| 192.168.56.103:7004 | 192.168.56.101:7001 |

This cross-VM placement means a single VM failure does not remove both members of the same primary-replica pair. However, the lab still has only three VM failure domains; losing a VM removes two Redis processes.

## Ports

Each Redis node uses a client port and a Cluster bus port. Redis derives the default bus port by adding 10,000 to the client port.

| Client port | Cluster bus port |
|---:|---:|
| 7000 | 17000 |
| 7001 | 17001 |
| 7002 | 17002 |
| 7003 | 17003 |
| 7004 | 17004 |
| 7005 | 17005 |

The client port accepts Redis commands. The Cluster bus port is used for internal node-to-node communication, including gossip, failure detection, and cluster coordination. Both port ranges must be reachable between cluster nodes over the private Host-only network.

## Instance configuration and service management

Configuration files follow this naming convention:

- `/etc/redis/redis-<PORT>.conf`
- Data directory: `/var/lib/redis/<PORT>/`
- Log file: `/var/log/redis/redis-<PORT>.log`
- Cluster node state file: `/var/lib/redis/<PORT>/nodes-<PORT>.conf`

The per-instance configuration includes the node's bind addresses, client port, data directory, log file, and Cluster settings. The Cluster-specific settings include:

```conf
cluster-enabled yes
cluster-config-file /var/lib/redis/<PORT>/nodes-<PORT>.conf
cluster-node-timeout 5000
cluster-announce-ip <VM_HOST_ONLY_IP>
cluster-announce-port <PORT>
cluster-announce-bus-port <PORT_PLUS_10000>
```

A systemd template unit, `redis-cluster@.service`, starts an instance using the port as the unit parameter and reads `/etc/redis/redis-%i.conf`.

Examples of the instance units:

| VM | Instance units |
|---|---|
| redis-vm1 | redis-cluster@7000, redis-cluster@7001 |
| redis-vm2 | redis-cluster@7002, redis-cluster@7003 |
| redis-vm3 | redis-cluster@7004, redis-cluster@7005 |

The package-provided standalone `redis.service` was disabled to avoid conflicting with the separately managed instances.

## Firewall

Redis client and Cluster bus ports were allowed for traffic from the private Host-only subnet `192.168.56.0/24`. The VM interfaces were observed in the `public` zone during the initial firewall inspection. The rules were scoped by source subnet; a later hardening pass can separate NAT and Host-only interfaces into dedicated zones after validating access requirements.

Required lab port ranges:

| TCP ports | Purpose | Source |
|---|---|---|
| 7000–7005 | Redis client/data traffic | Private lab subnet |
| 17000–17005 | Redis Cluster bus | Private lab subnet |

Do not expose these Redis ports to the public internet.

## Creating the Cluster

The cluster was created from VM1 with:

```bash
redis-cli --cluster create \
  192.168.56.101:7000 \
  192.168.56.102:7002 \
  192.168.56.103:7004 \
  192.168.56.101:7001 \
  192.168.56.102:7003 \
  192.168.56.103:7005 \
  --cluster-replicas 1
```

The proposed topology was reviewed and confirmed at the interactive prompt.

## Hash slots and sharding

Redis Cluster divides the keyspace into 16,384 hash slots, numbered 0 through 16,383. A key's hash slot determines which primary owns it. The created cluster distributed slots as follows:

| Primary | Assigned slots | Slot count |
|---|---:|---:|
| VM1:7000 | 0–5460 | 5,461 |
| VM2:7002 | 5461–10922 | 5,462 |
| VM3:7004 | 10923–16383 | 5,461 |

A key written through a non-owning node may produce a `MOVED` redirection. Use cluster-aware mode (`-c`) with `redis-cli` to follow these redirects automatically:

```bash
redis-cli -c -h 192.168.56.101 -p 7000 SET project "Redis HA Lab"
redis-cli -c -h 192.168.56.101 -p 7000 GET project
redis-cli -c -h 192.168.56.101 -p 7000 CLUSTER KEYSLOT project
```

The test key `project` was stored and retrieved successfully. Its hash slot was reported as 5904, which belongs to VM2 primary port 7002; the cluster-aware CLI followed the redirect.

## Cluster health verification

The following command was run against VM1 primary:

```bash
redis-cli -c -h 192.168.56.101 -p 7000 CLUSTER INFO | \
  grep -E "cluster_state|slots_assigned|slots_ok|known_nodes"
```

Observed output:

```text
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_known_nodes:6
```

The cluster creation output also confirmed that all nodes agreed about the slot configuration and that all 16,384 slots were covered.

## Replication verification status

The intended primary-replica pairs are listed above. A direct replication read test is still being completed. A test key initially returned a `MOVED` response when queried from VM2:7003 because the key's hash slot belonged to VM3:7004. This redirection is normal Cluster behavior and does not by itself indicate a replication failure.

To test replica reads for a key whose slot belongs to a different primary, use the replica's `READONLY` mode before reading:

```bash
redis-cli -h 192.168.56.102 -p 7003 READONLY
redis-cli -h 192.168.56.102 -p 7003 GET <KEY>
```

The replica read should be checked only after the primary write has propagated. The full three-pair replication test and failover test remain follow-up verification tasks.

## Operational notes

- Use `redis-cli -c` when issuing key commands through a Cluster endpoint so that `MOVED` redirects are followed.
- A direct connection without `-c` can return a redirection when the contacted node does not own the key's slot.
- `READONLY` on a replica permits replica reads for slots served by the cluster; it does not make the replica writable.
- Redis replication is asynchronous. A failover can lose writes that had not reached the replica.
- Redis Cluster's native failover is distinct from Redis Sentinel. Sentinel will be tested in a separate topology.

## Completion status

- [x] Six Redis instance configurations created across three VMs.
- [x] Six Cluster nodes started using systemd template instances.
- [x] Three-primary, three-replica Cluster created.
- [x] Replica placement across VMs confirmed from cluster creation/node output.
- [x] All 16,384 hash slots assigned.
- [x] `cluster_state:ok` and six known nodes verified.
- [x] Cluster-aware SET/GET test completed.
- [ ] Direct replication read verification completed for all three pairs.
- [ ] Controlled primary failure and automatic failover tested.

**Next:** Complete direct replication verification, then perform persistence and memory eviction experiments.
