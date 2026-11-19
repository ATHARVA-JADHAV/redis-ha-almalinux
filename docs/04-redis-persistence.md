# 04 - Redis Persistence: RDB and AOF

## Objective

Test Redis persistence on the Redis Cluster primary running on VM3 (`192.168.56.103:7004`). The experiments demonstrate how Redis writes data to disk and restores it after the server process is restarted.

> **Scope:** The tests documented here were performed on port `7004` only. They do not establish that persistence has been configured or verified on all six Cluster instances.

## Test environment

| Setting | Value |
|---|---|
| VM | redis-vm3 |
| Host-only IP | 192.168.56.103 |
| Redis instance | 7004 |
| Data directory | `/var/lib/redis/7004/` |
| RDB file | `dump.rdb` |
| AOF directory observed | `appendonlydir/` |
| Service | `redis-cluster@7004` |

Commands in the test were run locally on VM3 using `127.0.0.1`.

## 1. RDB snapshot test

RDB persistence stores point-in-time snapshots of the dataset in a binary dump file. In this test, two keys were written and a background snapshot was requested.

### Write test data

```bash
redis-cli -c -h 127.0.0.1 -p 7004 SET key_b "saved_in_snapshot"
redis-cli -c -h 127.0.0.1 -p 7004 SET key_d "second_snapshot_key"
```

Both SET operations returned `OK`.

### Trigger a background save

```bash
redis-cli -h 127.0.0.1 -p 7004 BGSAVE
```

Redis returned:

```text
Background saving started
```

### Inspect the snapshot file

```bash
ls -lh /var/lib/redis/7004/dump.rdb
```

The screenshot showed `dump.rdb` present in the instance data directory, with a reported size of 282 bytes.

### Restart and verify recovery

```bash
sudo systemctl restart redis-cluster@7004

redis-cli -c -h 127.0.0.1 -p 7004 GET key_b
redis-cli -c -h 127.0.0.1 -p 7004 GET key_d
```

After the restart, Redis returned the values:

```text
"saved_in_snapshot"
"second_snapshot_key"
```

**Result:** The two test keys were available after the instance restart, confirming recovery from the RDB snapshot for this test.

## 2. AOF persistence test

Append Only File (AOF) persistence records write operations so Redis can reconstruct the dataset by replaying the log during startup. Redis 7 and later use a multipart AOF layout; this instance showed an `appendonlydir` directory rather than requiring a single flat AOF file.

### Write a test key and inspect persistence status

```bash
redis-cli -c -h 127.0.0.1 -p 7004 SET aof_test_key "saved_in_aof_log"

ls -la /var/lib/redis/7004/

redis-cli -h 127.0.0.1 -p 7004 INFO persistence | \
  grep -E "aof_enabled|aof_current_size|aof_last_write_status"
```

The captured output showed:

```text
aof_enabled:1
aof_last_write_status:ok
aof_current_size:198
```

The instance data directory contained `appendonlydir`.

### Restart and verify recovery

```bash
sudo systemctl restart redis-cluster@7004

redis-cli -c -h 127.0.0.1 -p 7004 GET aof_test_key
```

Redis returned:

```text
"saved_in_aof_log"
```

**Result:** The AOF test key was available after restarting the Redis instance.

## 3. Persist the AOF configuration change

The AOF setting was explicitly enabled and the instance configuration was rewritten:

```bash
redis-cli -h 127.0.0.1 -p 7004 CONFIG SET appendonly yes
redis-cli -h 127.0.0.1 -p 7004 CONFIG REWRITE
```

Both commands returned `OK`.

`CONFIG SET` changes the running server configuration. `CONFIG REWRITE` writes supported configuration changes back to the instance's configuration file so they can survive a later restart. The successful command output was captured; a further restart followed by checking `CONFIG GET appendonly` is a useful additional confirmation of the rewritten setting.

## RDB and AOF comparison

| Aspect | RDB | AOF |
|---|---|---|
| Mechanism | Point-in-time dataset snapshot | Records write operations for replay |
| Test command | `BGSAVE` | `CONFIG SET appendonly yes` |
| Files observed | `dump.rdb` | `appendonlydir/` |
| Recovery test | Keys survived service restart | Test key survived service restart |
| Main trade-off | Compact snapshots; writes since the last snapshot may be lost | Better durability potential depending on fsync policy; more disk I/O and log management |

The actual recovery point and durability guarantees depend on the configured RDB save schedule and AOF fsync policy. Those policies should be inspected explicitly before making a production-style durability claim.

## Completion status

- [x] RDB test keys written on VM3 port 7004.
- [x] `BGSAVE` completed and `dump.rdb` observed.
- [x] RDB test keys retrieved after restarting the instance.
- [x] AOF enabled status and successful last-write status observed.
- [x] AOF test key retrieved after restarting the instance.
- [x] `CONFIG SET appendonly yes` and `CONFIG REWRITE` returned `OK`.
- [ ] Inspect and record the exact RDB save schedule and AOF fsync policy.
- [ ] Decide and apply persistence configuration consistently across the six instances, if required.
- [ ] Add screenshots to the repository's evidence directory.

## Next: memory management and eviction

The next experiment will set a controlled `maxmemory` limit and use `allkeys-lru` to observe key eviction when the instance reaches its memory limit. This will be performed as a separate controlled test, with eviction statistics checked before and after the workload.
