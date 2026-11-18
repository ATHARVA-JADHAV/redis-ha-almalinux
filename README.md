# Redis High Availability Lab on AlmaLinux

A hands-on Linux project to build, configure, secure, and test a Redis-based in-memory data store across multiple AlmaLinux virtual machines.

The goal is to understand not just the configuration, but also the networking, persistence, replication, failover, security, and troubleshooting involved in operating Redis.

## Current lab status

- Three AlmaLinux 9.8 VirtualBox VMs provisioned with static Host-only addresses.
- NAT and Host-only networking configured; VM-to-VM connectivity verified.
- System package updates completed successfully on all three VMs.
- Redis installed on all three VMs using the official Redis RPM repository.
- Redis multi-instance configuration and Cluster formation are the next implementation stages.

Redis is installed as software; the six-node Cluster has **not** yet been configured or created.

## Lab environment

- **OS:** AlmaLinux 9.8 (VirtualBox)
- **VMs:** 3 Linux virtual machines
- **Redis deployment modes:** Redis Cluster and Redis Sentinel (implemented and tested separately)
- **Networking:** NAT for outbound package access; Host-only network for private VM-to-VM traffic

| VM | Hostname | Host-only address |
|---|---|---|
| VM 1 | redis-vm1 | 192.168.56.101/24 |
| VM 2 | redis-vm2 | 192.168.56.102/24 |
| VM 3 | redis-vm3 | 192.168.56.103/24 |

The Host-only network is 192.168.56.0/24, the host address is 192.168.56.1, and its DHCP server is disabled.

## What we will build

### 1. Redis Cluster

- Run six independent Redis server processes across the three VMs.
- Create a cluster with three primary nodes and three replicas.
- Understand hash slots, key distribution, redirects, and replica assignment.
- Test node loss and Redis Cluster's native failover behavior.

### 2. Persistence and memory management

- Configure and test RDB snapshots.
- Configure and test AOF persistence.
- Set a memory limit and test the `allkeys-lru` eviction policy.
- Verify behavior using Redis commands and logs.

### 3. Redis Sentinel

- Build a separate master-replica deployment monitored by three Sentinel processes.
- Configure quorum and automatic failover.
- Stop the primary and observe promotion of a replica.
- Verify how clients discover the new primary.

> Redis Cluster and Sentinel are separate deployment modes in this lab. Sentinel will not be used to manage Redis Cluster failover.

### 4. Security

- Configure Redis ACL users and authentication.
- Restrict network access with firewalld.
- Configure TLS for encrypted connections.
- Test mutual TLS (client certificates) if included in the final lab scope.

## Implementation approach

We will work through the lab in small, verifiable stages:

1. Provision the VMs and verify private connectivity.
2. Update the operating systems and verify NAT internet access.
3. Install Redis and inspect the package, service, and configuration paths.
4. Configure six independent Redis instances (two per VM).
5. Configure firewall access and form the three-primary, three-replica Cluster.
6. Verify cluster health, hash-slot coverage, and replication.
7. Test RDB, AOF, and memory eviction.
8. Reconfigure the lab as a separate Sentinel topology and test failover.
9. Apply ACL authentication and TLS/mTLS, then repeat relevant tests.

Each stage will be completed and checked before moving on. Configuration files and scripts will be added only when we reach the stage that needs them.

## Repository layout

The repository will grow as the lab progresses. Documentation, configuration examples, scripts, and tests will be added incrementally rather than creating everything upfront.

## Important

This is a learning lab, not a production-ready deployment. Passwords, certificates, firewall rules, and resource limits will be chosen for the lab and clearly documented. Never commit real passwords, private keys, or other secrets to this repository.
