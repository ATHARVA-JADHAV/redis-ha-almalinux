# Redis High Availability Lab on AlmaLinux

A hands-on Linux project to build, configure, secure, and test a Redis-based in-memory data store across multiple AlmaLinux virtual machines.

The goal is to understand not just the configuration, but also the networking, persistence, replication, failover, security, and troubleshooting involved in operating Redis.

## Lab environment

- **OS:** AlmaLinux (VirtualBox)
- **VMs:** 3 Linux virtual machines
- **Redis deployment modes:** Redis Cluster and Redis Sentinel (implemented and tested separately)
- **Networking:** Private virtual network with static IP addresses

Planned VM addresses (we will confirm these during setup):

| VM | Planned address |
|---|---|
| redis-vm1 | 192.168.56.101 |
| redis-vm2 | 192.168.56.102 |
| redis-vm3 | 192.168.56.103 |

## What we will build

### 1. Redis Cluster

- Run multiple Redis processes across the three VMs.
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

1. Prepare the VMs and confirm their network connectivity.
2. Install Redis and verify the service.
3. Configure and test Redis Cluster.
4. Test persistence and memory eviction.
5. Switch to a separate Sentinel topology and test failover.
6. Apply authentication and TLS, then repeat the relevant tests.
7. Document commands, results, and troubleshooting.

Each stage will be completed and checked before moving on. Configuration files and scripts will be added only when we reach the stage that needs them.

## Repository layout

The repository will grow as the lab progresses. We will add documentation, configuration examples, scripts, and tests incrementally rather than creating everything upfront.

## Important

This is a learning lab, not a production-ready deployment. Passwords, certificates, firewall rules, and resource limits will be chosen for the lab and clearly documented. Never commit real passwords, private keys, or other secrets to this repository.
