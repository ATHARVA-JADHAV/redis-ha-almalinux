# 05 - Redis Memory Management and LRU Eviction

## Objective

Demonstrate Redis memory limits and Least Recently Used (LRU) eviction using the `allkeys-lru` policy. The experiment was performed on primary VM1:7000 and its replica VM2:7003.

## 1. Baseline and test-node selection

The initial inspection found the following baseline on the inspected instance:

| Setting | Initial value |
|---|---:|
| `used_memory` | 2,381,920 bytes (approximately 2.27 MiB) |
| `maxmemory` | 0 (no configured limit) |
| `maxmemory-policy` | `noeviction` |
| `evicted_keys` | 0 |

During topology verification, port 7004 was found to be a replica in the then-current topology. The topology was subsequently rebalanced. The final LRU experiment was therefore moved to VM1 primary port 7000, whose replica is VM2 port 7003.

A temporary 16 MiB / `allkeys-lru` configuration previously present on VM3:7004 was reset to `maxmemory 0` and `maxmemory-policy noeviction` before the final experiment.

## 2. Final topology relevant to the experiment

| Role | Instance |
|---|---|
| Primary being tested | VM1:7000 (192.168.56.101) |
| Replica | VM2:7003 (192.168.56.102) |
| Hash slot for test tag | 4402 |
| Hash tag | `lru_v1_2` |

Slot 4402 is within VM1:7000's assigned range, 0–5460. Using the same Redis hash tag in each test key ensures the keys map to the same slot and are served by the selected primary.

## 3. Configure the memory limit and eviction policy

On VM1, the primary was configured with a 16 MiB limit and `allkeys-lru`:

```bash
redis-cli -h 127.0.0.1 -p 7000 CONFIG SET maxmemory 16mb
redis-cli -h 127.0.0.1 -p 7000 CONFIG SET maxmemory-policy allkeys-lru
```

The baseline before the load was reported as:

```text
used_memory_human:2.57M
maxmemory_human:16.00M
evicted_keys:0
```

The other instance settings should not be assumed to have changed: this controlled eviction test was performed on primary 7000. The report specifically records that 7004 was reset to `maxmemory 0` and `noeviction`.

## 4. Generate a workload

A hash tag was selected whose key slot belonged to VM1:7000:

```text
TAG=lru_v1_2
SLOT=4402
```

The test generated 5,000 SET commands, each containing an approximately 4 KiB payload, and piped them to Redis using `redis-cli --pipe`. All test keys used the same hash tag so they were routed to the same Cluster slot.

Representative command pattern:

```bash
VAL=$(printf 'A%.0s' {1..4096})
for i in $(seq 1 5000); do
  printf 'SET "{lru_v1_2}:bulk:%s" "%s"\r\n' "$i" "$VAL"
done | redis-cli -c -h 127.0.0.1 -p 7000 --pipe
```

## 5. Observed eviction results

After the workload, the reported measurements were:

| Metric | Result |
|---|---:|
| `used_memory_human` | 16.00M |
| `maxmemory_human` | 16.00M |
| `evicted_keys` | 6,497 |
| Post-eviction SET | `OK` |
| Post-eviction GET | `write_ok` |

The test demonstrates that Redis evicted keys to remain within the configured memory limit and continued accepting writes. The eviction counter is cumulative for the running server; it is not reset by manually deleting test keys.

LRU is an approximation in Redis, so it should not be interpreted as a strict guarantee that the single oldest key will always be evicted first.

## 6. Verify replica state

The corresponding replica VM2:7003 was inspected:

```bash
redis-cli -h 127.0.0.1 -p 7003 INFO replication
redis-cli -c -h 127.0.0.1 -p 7003 GET "{lru_v1_2}:after_evict"
```

Reported replica details included:

```text
role:slave
master_host:192.168.56.101
master_port:7000
master_link_status:up
```

The replica returned `write_ok` for the post-eviction test key. This confirms the test write was replicated and readable from the replica.

A replica normally follows the primary's dataset and does not independently apply its own maxmemory eviction policy to diverge from the primary. The eviction experiment is therefore measured on the primary, with the replica checked for replication consistency.

## 7. Useful verification commands

```bash
# Memory configuration
redis-cli -h 127.0.0.1 -p 7000 CONFIG GET maxmemory
redis-cli -h 127.0.0.1 -p 7000 CONFIG GET maxmemory-policy

# Memory usage and eviction count
redis-cli -h 127.0.0.1 -p 7000 INFO memory
redis-cli -h 127.0.0.1 -p 7000 INFO stats

# Replication status on replica
redis-cli -h 127.0.0.1 -p 7003 INFO replication
```

## 8. Scope and remaining operational decision

The recorded 16 MiB / `allkeys-lru` experiment applies to primary VM1:7000. It should not be described as a cluster-wide memory configuration. The other five processes have independent configuration and memory limits.

Before treating the lab as production-like, decide whether the same memory policy should be configured on each primary and how much memory should be reserved for the operating system, replication buffers, client buffers, and process overhead. Do not set the Redis limit equal to the VM's total RAM.

## Completion status

- [x] Initial memory settings inspected.
- [x] Test tag selected in the hash-slot range owned by primary VM1:7000.
- [x] 16 MiB memory limit configured on VM1:7000.
- [x] `allkeys-lru` configured on VM1:7000.
- [x] Workload generated through `redis-cli --pipe`.
- [x] Eviction observed (`evicted_keys:6497`).
- [x] Redis continued accepting writes after eviction.
- [x] Replica VM2:7003 returned the post-eviction test value.
- [ ] Decide whether to standardize memory limits/policy across all primary and replica processes.
- [ ] Add terminal screenshots/evidence to the repository.

**Next:** Capture the final evidence and complete the Member A sign-off. Member B can then begin as a separate Sentinel and security deployment.
