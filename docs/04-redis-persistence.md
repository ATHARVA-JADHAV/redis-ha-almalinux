# 04 - Redis Persistence: RDB and AOF

## Objective

Test Redis persistence and verify the persistence configuration across the six Redis Cluster instances. The initial hands-on RDB/AOF restart tests were performed on VM3 port 7004. A later audit inspected the persistence settings on all six ports, followed by a recovery test involving primary VM1:7000 and replica VM2:7003.

## 1. Initial RDB snapshot test on VM3:7004

RDB persistence stores point-in-time snapshots of the dataset in a binary dump file.

Test data was written and a background save was requested:

```bash
redis-cli -c -h 127.0.0.1 -p 7004 SET key_b "saved_in_snapshot"
redis-cli -c -h 127.0.0.1 -p 7004 SET key_d "second_snapshot_key"
redis-cli -h 127.0.0.1 -p 7004 BGSAVE
ls -lh /var/lib/redis/7004/dump.rdb
```

The `BGSAVE` command returned `Background saving started`. The screenshot showed `dump.rdb` present in the instance data directory. After restarting `redis-cluster@7004`, both test values were retrieved successfully.

## 2. Initial AOF test on VM3:7004

Append Only File (AOF) persistence records write operations so Redis can reconstruct the dataset during startup. The instance used the multipart AOF layout under `appendonlydir/`.

The test key was written and persistence status inspected:

```bash
redis-cli -c -h 127.0.0.1 -p 7004 SET aof_test_key "saved_in_aof_log"
ls -la /var/lib/redis/7004/
redis-cli -h 127.0.0.1 -p 7004 INFO persistence
```

The captured status included:

```text
aof_enabled:1
aof_last_write_status:ok
aof_current_size:198
```

After restarting `redis-cluster@7004`, the key was retrieved successfully.

AOF was explicitly enabled and the configuration rewritten:

```bash
redis-cli -h 127.0.0.1 -p 7004 CONFIG SET appendonly yes
redis-cli -h 127.0.0.1 -p 7004 CONFIG REWRITE
```

Both commands returned `OK`.

## 3. Persistence configuration audit across all six instances

The later audit inspected `save`, `appendonly`, `appendfsync`, and `dir` on ports 7000–7005.

The reported settings were consistent across all six instances:

| Setting | Reported value | Meaning |
|---|---|---|
| `save` | `3600 1 300 100 60 10000` | RDB snapshot rules are configured |
| `appendonly` | `yes` | AOF enabled |
| `appendfsync` | `everysec` | Redis fsyncs the AOF approximately once per second |
| `dir` | `/var/lib/redis/<PORT>` | Each instance uses its own data directory |

The RDB save rule contains multiple time/change thresholds. The AOF `everysec` policy provides a balance between durability and I/O overhead; it is not an absolute zero-data-loss guarantee.

## 4. Recovery and replica restart test

A recovery marker was written to primary VM1:7000 using a hash tag belonging to that primary's slot range:

```bash
redis-cli -c -h 127.0.0.1 -p 7000 SET "{lru_v1_2}:recovery_check" "survives_restart"
redis-cli -h 127.0.0.1 -p 7000 BGSAVE
```

The marker was checked on replica VM2:7003. The replica was then restarted and checked again.

The report recorded the following sequence:

1. The replica returned `PONG` after restart.
2. The recovery marker was still readable.
3. The replication link briefly showed `down` while reconnecting.
4. A subsequent check showed `master_link_status:up`.
5. The final Cluster check reported `cluster_state:ok`.

The restart command used during the experiment attempted several service/process command alternatives. For repeatable operations, use the known systemd instance unit `redis-cluster@7003` and verify its status before and after restart.

## RDB and AOF comparison

| Aspect | RDB | AOF |
|---|---|---|
| Mechanism | Point-in-time dataset snapshot | Records write operations for replay |
| Test command | `BGSAVE` | `CONFIG SET appendonly yes` |
| Files observed | `dump.rdb` | `appendonlydir/` |
| Recovery test | Keys survived instance restart | Test key survived instance restart |
| Trade-off | Compact snapshots; writes since the last snapshot may be lost | More granular durability depending on fsync policy; additional disk I/O and log management |

## Completion status

- [x] RDB snapshot created and observed on VM3:7004.
- [x] RDB test keys retrieved after restarting VM3:7004.
- [x] AOF enabled and successful write status observed on VM3:7004.
- [x] AOF test key retrieved after restarting VM3:7004.
- [x] AOF configuration rewrite returned `OK`.
- [x] RDB save schedule, AOF enablement and fsync policy audited across all six instances (as reported).
- [x] Recovery marker verified on primary and replica.
- [x] Replica restart and replication-link reconnection verified.
- [ ] Add terminal screenshots/evidence to the repository.

**Next:** Continue with the memory-management experiment documented in [05 - Redis Memory Management](05-redis-memory-management.md).
