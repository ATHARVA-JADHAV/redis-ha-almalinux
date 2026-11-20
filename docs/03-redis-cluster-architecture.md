# 03 - Redis Cluster Architecture and Implementation

## Objective

Deploy and operate a six-node Redis Cluster across three AlmaLinux 9.8 virtual machines. The final topology contains three primaries and three replicas, with one primary and one replica process on each VM and each replica placed on a different VM from its primary.

## Lab topology

| VM | Host-only address | Redis port | Final role |
|---|---|---:|---|
| redis-vm1 | 192.168.56.101 | 7000 | Primary |
| redis-vm1 | 192.168.56.101 | 7001 | Replica of VM3:7004 |
| redis-vm2 | 192.168.56.102 | 7002 | Primary |
| redis-vm2 | 192.168.56.102 | 7003 | Replica of VM1:7000 |
| redis-vm3 | 192.168.56.103 | 7004 | Primary |
| redis-vm3 | 192.168.56.103 | 7005 | Replica of VM2:7002 |

## Final primary-replica assignments

| Primary | Replica | Replica's host |
|---|---|---|
| 192.168.56.101:7000 | 192.168.56.102:7003 | VM2 |
| 192.168.56.102:7002 | 192.168.56.103:7005 | VM3 |
| 192.168.56.103:7004 | 192.168.56.101:7001 | VM1 |

### Topology balancing operation

An initial inspection found an unbalanced role distribution: VM1 hosted primaries 7000 and 7001, VM2 hosted primary 7002 and replica 7003, while VM3 hosted replicas 7004 and 7005.

The topology was rebalanced using the following operations:

1. On VM3, port 7004 was promoted using a Cluster failover operation.
2. On VM1, port 7001 was configured to replicate from the promoted VM3:7004 node.
3. Cluster state and node roles were inspected again.

Commands recorded for the operation:

```bash
# On VM3
redis-cli -h 127.0.0.1 -p 7004 CLUSTER FAILOVER

# On VM1
redis-cli -h 127.0.0.1 -p 7001 CLUSTER REPLICATE f82c4e90e261be4f0edad1ac3d501e70191f296d

# Verify
redis-cli -h 192.168.56.101 -p 7000 CLUSTER INFO
redis-cli -h 192.168.56.101 -p 7000 CLUSTER NODES
```

The resulting topology reported VM3:7004 as primary for slots 10923–16383 and VM1:7001 as its replica. The final topology is balanced by process role and distributes each primary's replica to another VM.

## Hash slots and sharding

Redis Cluster divides the keyspace into 16,384 hash slots, numbered 0 through 16,383. A key's hash slot determines which primary owns it.

| Primary | Assigned slots | Slot count |
|---|---:|---:|
| VM1:7000 | 0–5460 | 5,461 |
| VM2:7002 | 5461–10922 | 5,462 |
| VM3:7004 | 10923–16383 | 5,461 |

A key written through a non-owning node may produce a `MOVED` redirection. Use cluster-aware mode (`-c`) with `redis-cli` to follow these redirects automatically.

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

The per-instance configuration includes bind addresses, client port, data directory, log file, and Cluster settings. Key Cluster settings include:

```conf
cluster-enabled yes
cluster-config-file /var/lib/redis/<PORT>/nodes-<PORT>.conf
cluster-node-timeout 5000
cluster-announce-ip <VM_HOST_ONLY_IP>
cluster-announce-port <PORT>
cluster-announce-bus-port <PORT_PLUS_10000>
```

A systemd template unit, `redis-cluster@.service`, starts an instance using the port as the unit parameter and reads `/etc/redis/redis-%i.conf`. The package-provided standalone `redis.service` was disabled to avoid conflicting with these separately managed instances.

## Firewall

Redis client and Cluster bus ports were allowed for traffic from the private Host-only subnet `192.168.56.0/24`. The VM interfaces were observed in the `public` zone during the initial firewall inspection. Rules were scoped by source subnet. Do not expose Redis ports to the public internet.

| TCP ports | Purpose | Source |
|---|---|---|
| 7000–7005 | Redis client/data traffic | Private lab subnet |
| 17000–17005 | Redis Cluster bus | Private lab subnet |

## Cluster creation

The initial Cluster was created from VM1 with:

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

The topology was subsequently rebalanced as described above.

## Cluster health verification

The following checks were used:

```bash
redis-cli -c -h 192.168.56.101 -p 7000 CLUSTER INFO
redis-cli -c -h 192.168.56.101 -p 7000 CLUSTER NODES
```

The earlier captured health output showed:

```text
cluster_state:ok
cluster_slots_assigned:16384
cluster_slots_ok:16384
cluster_known_nodes:6
```

After the topology rebalance, the report again recorded `cluster_state:ok` and the final node-role assignments listed above.

## Replication and recovery verification

The final topology's replication checks reported:

- VM1:7000 is primary; VM2:7003 is its replica, with `master_link_status:up`.
- VM2:7002 is primary; VM3:7005 is its replica.
- VM3:7004 is primary; VM1:7001 is its replica.

A recovery marker was written through primary VM1:7000 and read from replica VM2:7003. Replica 7003 was restarted, returned `PONG`, retained the replicated recovery marker, and its replication link returned to `up` after reconnecting. The final Cluster check reported `cluster_state:ok`.

### Cluster-native primary failover exercise

A controlled failover exercise was subsequently reported using VM1:7000 and its replica VM2:7003:

1. Cluster health and node roles were inspected before the test.
2. A test key using the hash tag `{lru_v1_2}` was written through VM1:7000 and checked on VM2:7003.
3. The primary instance was stopped on VM1 with `sudo systemctl stop redis-cluster@7000`.
4. The Cluster was inspected from VM2:7003 using `CLUSTER INFO` and `CLUSTER NODES`.
5. VM2:7003 was reported promoted to primary.
6. The pre-failover test key was checked through the promoted node, and a new write was reported successful.

This exercise demonstrates Redis Cluster's native failover path: the replica can be promoted when its primary becomes unavailable and the remaining Cluster masters authorize the failover. Redis replication is asynchronous, so writes not yet replicated before the outage are not guaranteed to survive.

The available test report covers promotion and post-promotion reads/writes. Restarting the former primary VM1:7000 and verifying that it rejoins as a replica should be recorded separately once that recovery check has been performed.

## Operational notes

- Use `redis-cli -c` when issuing key commands through a Cluster endpoint so that `MOVED` redirects are followed.
- A direct connection without `-c` can return a redirection when the contacted node does not own the key's slot.
- `READONLY` on a replica permits replica reads for slots served by the Cluster; it does not make the replica writable.
- Redis replication is asynchronous. A failover can lose writes that had not reached the replica.
- Redis Cluster's native failover is distinct from Redis Sentinel. Sentinel will be tested in a separate topology.
- The three-VM lab has three VM failure domains. Each VM contains two Redis processes, so losing a VM removes both processes hosted there.

## Completion status

- [x] Six Redis instance configurations created across three VMs.
- [x] Three-primary, three-replica Cluster created.
- [x] One primary and one replica process per VM in the final topology.
- [x] Each replica placed on a different VM from its primary.
- [x] All 16,384 hash slots assigned.
- [x] Cluster health reported as `cluster_state:ok` with six known nodes.
- [x] Final primary-replica links inspected.
- [x] Recovery marker survived replica restart and replication reconnected.
- [x] Controlled primary outage tested for VM1:7000.
- [x] VM2:7003 reported promoted to primary after VM1:7000 stopped.
- [x] Pre-failover test key and a new write checked through the promoted node.
- [ ] Restart former primary VM1:7000 and verify its rejoin/replica role.
- [ ] Commit final screenshots/evidence to the repository.

**Next:** Complete the former-primary rejoin check and evidence commit. Member B is implemented separately using Redis Sentinel; Sentinel does not manage this Cluster.
