# Redis High Availability Lab on AlmaLinux

A hands-on Linux project to build, configure, secure, and test Redis high availability across multiple AlmaLinux virtual machines.

The goal is to understand networking, persistence, memory management, replication, failover, security, and troubleshooting involved in operating Redis.

## Current lab status

- Three AlmaLinux 9.8 VirtualBox VMs provisioned with static Host-only addresses.
- NAT and Host-only networking configured; VM-to-VM connectivity verified.
- Redis installed on all three VMs using the official Redis RPM repository.
- Six Redis Cluster instances deployed across three VMs.
- Three-primary, three-replica Cluster balanced with one primary and one replica process per VM.
- Cluster health verified with all 16,384 hash slots assigned and six known nodes.
- Cross-VM primary-replica links verified.
- RDB snapshots and AOF persistence enabled and audited across all six instances.
- AOF fsync policy reported as `everysec` across all six instances.
- RDB/AOF restart recovery tested; replica restart and replication reconnection verified.
- `allkeys-lru` eviction demonstrated on primary VM1:7000 with a 16 MiB memory limit.
- Cluster-native failover exercise reported: VM1:7000 was stopped, VM2:7003 was promoted, and the test key plus a new write were checked through the promoted node.
- Remaining Member A close-out: restart former primary VM1:7000 and verify its rejoin as a replica; commit final screenshots/evidence.
- Member B Sentinel HA deployment completed on Redis port 7100 and Sentinel port 26379. Three Sentinels are running with quorum 2; automatic failover from VM1 to VM3, post-failover writes, and VM1 recovery as a replica were verified. See the Sentinel deployment and failover runbook.

## Lab environment

- **OS:** AlmaLinux 9.8 (VirtualBox)
- **VMs:** 3 Linux virtual machines
- **Redis deployment modes:** Redis Cluster and Redis Sentinel (separate lab topologies)
- **Networking:** NAT for outbound package access; Host-only network for private VM-to-VM traffic

| VM | Hostname | Host-only address |
|---|---|---|
| VM 1 | redis-vm1 | 192.168.56.101/24 |
| VM 2 | redis-vm2 | 192.168.56.102/24 |
| VM 3 | redis-vm3 | 192.168.56.103/24 |

The Host-only network is 192.168.56.0/24, the host address is 192.168.56.1, and its DHCP server is disabled.

## Final Redis Cluster topology

Each VM runs two independent Redis processes. Client ports are 7000–7005; corresponding Cluster bus ports are 17000–17005.

| VM | Primary | Replica |
|---|---:|---:|
| redis-vm1 (192.168.56.101) | 7000 | 7001 |
| redis-vm2 (192.168.56.102) | 7002 | 7003 |
| redis-vm3 (192.168.56.103) | 7004 | 7005 |

Cross-VM replication assignments:

| Primary | Replica |
|---|---|
| VM1:7000 | VM2:7003 |
| VM2:7002 | VM3:7005 |
| VM3:7004 | VM1:7001 |

Hash slot allocation:

| Primary | Hash slots |
|---|---:|
| VM1:7000 | 0–5460 |
| VM2:7002 | 5461–10922 |
| VM3:7004 | 10923–16383 |

The final role balance was achieved by promoting VM3:7004 to primary for slots 10923–16383 and reconfiguring VM1:7001 as its replica. The final replica links were reported connected.

## Project scope

### Member A — Redis Cluster, replication, persistence and memory
- Six Redis instances across three VMs; three primaries and three replicas.
- Hash slots, sharding, redirections, and cross-VM replication.
- RDB snapshots and AOF persistence.
- Memory limits and `allkeys-lru` eviction.
- Restart recovery, replication reconnection, and Cluster-native primary failover.

### Member B — High availability and security
- Separate Redis Sentinel master-replica topology.
- Sentinel quorum and automatic failover.
- Client discovery of the promoted primary.
- Redis ACL authentication.
- TLS encryption for data in transit and mutual TLS where required.

> Redis Cluster and Sentinel are separate deployment modes in this lab. Sentinel does not manage Redis Cluster's native failover.

## Documentation

| Document | Coverage |
|---|---|
| [VM provisioning](docs/01-vm-provisioning.md) | VirtualBox networking, static IPs, OS setup |
| [Redis installation](docs/02-redis-installation.md) | Official repository, package installation and service inspection |
| [Cluster architecture](docs/03-redis-cluster-architecture.md) | Final balanced six-node topology, slots, replication and health |
| [Persistence](docs/04-redis-persistence.md) | RDB/AOF tests, six-instance settings audit and recovery |
| [Memory management](docs/05-redis-memory-management.md) | maxmemory, allkeys-lru eviction, workload and replica verification |
| [Redis Sentinel high availability](docs/06-redis-sentinel-ha.md) | Separate primary-replica deployment, Sentinel configuration, authentication, quorum, failover results and troubleshooting |\n| [Redis Sentinel failover and recovery runbook](docs/07-redis-sentinel-failover-testing.md) | Step-by-step per-VM preflight, controlled primary failure, promotion verification, post-failover writes, former-primary recovery and troubleshooting |

## Implementation approach

Each stage is verified before moving forward. Configuration files, scripts, terminal output, and screenshots will be added incrementally as the lab progresses.

## Important limitations

This is a learning lab, not a production-ready deployment. The three-VM topology has only three VM failure domains: losing one VM also loses both Redis processes running on it. Redis replication is asynchronous and does not guarantee zero data loss. The LRU experiment was performed on one primary; it does not imply that the same memory limit has been configured on all six instances.

Never commit real passwords, private keys, or other secrets to this repository.
