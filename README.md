# Redis High Availability Lab on AlmaLinux

A hands-on Linux project to build, configure, secure, and test Redis high availability across multiple AlmaLinux virtual machines.

The goal is to understand the networking, persistence, memory management, replication, failover, security, and troubleshooting involved in operating Redis.

## Current lab status

- Three AlmaLinux 9.8 VirtualBox VMs provisioned with static Host-only addresses.
- NAT and Host-only networking configured; VM-to-VM connectivity verified.
- Redis installed on all three VMs using the official Redis RPM repository.
- Six Redis Cluster instances configured across the three VMs.
- Three-primary, three-replica Redis Cluster created.
- Cluster health verified: `cluster_state:ok`, all 16,384 slots assigned, six known nodes.
- Basic cluster-aware SET/GET test completed.
- RDB and AOF persistence experiments performed on VM3 Redis instance `7004`; restart recovery was verified for test keys.
- Memory limit and LRU eviction testing are the next stage.

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

## Redis Cluster topology

Each VM runs two independent Redis processes. The client ports are 7000–7005 and the corresponding Cluster bus ports are 17000–17005.

| VM | Primary | Replica |
|---|---|---|
| redis-vm1 | 7000 | 7001 |
| redis-vm2 | 7002 | 7003 |
| redis-vm3 | 7004 | 7005 |

Replica assignments selected during cluster creation:

| Primary | Replica |
|---|---|
| 192.168.56.101:7000 | 192.168.56.102:7003 |
| 192.168.56.102:7002 | 192.168.56.103:7005 |
| 192.168.56.103:7004 | 192.168.56.101:7001 |

The verified primary slot ranges are:

| Primary | Hash slots |
|---|---:|
| 192.168.56.101:7000 | 0–5460 |
| 192.168.56.102:7002 | 5461–10922 |
| 192.168.56.103:7004 | 10923–16383 |

The cluster reported `cluster_state:ok`, `cluster_slots_assigned:16384`, `cluster_slots_ok:16384`, and `cluster_known_nodes:6`.

## What this project covers

### 1. Redis Cluster
- Six independent Redis server processes across three VMs.
- Three primaries and three replicas.
- Hash slots, sharding, client redirections, and replication.
- Controlled primary failure and native Cluster failover testing.

### 2. Persistence and memory management
- RDB snapshot testing with `BGSAVE` and restart recovery.
- AOF testing, including AOF status inspection and restart recovery.
- Memory limits and `allkeys-lru` eviction behavior.

The initial RDB and AOF experiments were performed on Redis port `7004` on VM3. These tests do not mean persistence settings have been applied and verified on all six instances.

### 3. Redis Sentinel
- A separate master-replica deployment monitored by Sentinel processes.
- Quorum and automatic failover.
- Client discovery of the promoted primary.

> Redis Cluster and Sentinel are separate deployment modes in this lab. Sentinel will not manage Redis Cluster failover.

### 4. Security
- Redis ACL users and authentication.
- Network restrictions with firewalld.
- TLS and, if included in the final scope, mutual TLS.

## Documentation

| Document | Coverage |
|---|---|
| [VM provisioning](docs/01-vm-provisioning.md) | VirtualBox networking, static IPs, OS setup |
| [Redis installation](docs/02-redis-installation.md) | Official repository, package installation and service inspection |
| [Cluster architecture](docs/03-redis-cluster-architecture.md) | Actual six-node topology, slots, replication and Cluster concepts |
| [Persistence](docs/04-redis-persistence.md) | RDB and AOF experiments on port 7004 |

## Implementation approach

Each stage is verified before moving forward. Configuration files, scripts, terminal output, and screenshots will be added incrementally as the lab progresses.

## Important

This is a learning lab, not a production-ready deployment. The three-VM topology has only three VM failure domains: losing one VM also loses both Redis processes running on it. Redis replication is asynchronous and does not guarantee zero data loss.

Never commit real passwords, private keys, or other secrets to this repository.
