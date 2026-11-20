# Redis Sentinel Failover and Recovery Runbook

## 1. Purpose

This runbook explains how to perform a controlled Redis Sentinel automatic-failover exercise, verify the promoted primary, test writes after promotion, and safely return the failed node as a replica.

It is written for the three-VM AlmaLinux 9.8 lab. Commands are grouped by the VM on which they must run.

**Important:** This is a lab procedure. It deliberately stops a Redis data service and causes a short interruption. Do not run it against production data.

## 2. Lab topology

### Redis data instances

| VM | Host-only IP | Redis data port | Sentinel port | Initial role used in the recorded test |
|---|---|---:|---:|---|
| redis-vm1 | 192.168.56.101 | 7100 | 26379 | Primary |
| redis-vm2 | 192.168.56.102 | 7100 | 26379 | Replica |
| redis-vm3 | 192.168.56.103 | 7100 | 26379 | Replica |

Sentinel monitors the master group named `mymaster` with quorum `2`. There is one Sentinel process on each VM.

### Service names

| Service | Purpose |
|---|---|
| `redis-sentinel-lab.service` | Redis data server on port 7100. Despite its name, this is **not** the Sentinel process. |
| `redis-sentinel.service` | Redis Sentinel process on port 26379. |

Configuration and log paths:

- Redis data config: `/etc/redis/redis-7100.conf`
- Sentinel config: `/var/lib/redis/sentinel/sentinel.conf`
- Redis data log: `/var/log/redis/redis-7100.log`
- Sentinel log: `/var/log/redis/sentinel.log`

The Redis Cluster deployment (ports 7000–7005 and bus ports 17000–17005) is separate. **Do not stop, reconfigure, or use Cluster services for this Sentinel test.**

## 3. Safety and prerequisites

Before starting:

1. Confirm that all three VMs are reachable on the Host-only network.
2. Confirm that the Redis data service and Sentinel service are running on all three VMs.
3. Confirm that Sentinel has quorum and sees the current primary and replicas.
4. Confirm that the current primary and both replicas are healthy.
5. Use a disposable test key. Do not use important application data.
6. Have the Redis data password and Sentinel password available locally. Never put passwords in commands, shell history, screenshots, Git commits, or shared logs. Use `redis-cli --askpass` and redact any accidental exposure.

The commands below assume the current Sentinel master group is named `mymaster`.

## 4. Pre-failover checks

### 4.1 Run on each VM: check both services

Run on **VM1, VM2, and VM3**:

```bash
sudo systemctl is-active redis-sentinel-lab.service
sudo systemctl is-active redis-sentinel.service
```

Expected: both commands return `active` on each VM.

If a service is not active, inspect it before proceeding:

```bash
sudo systemctl status redis-sentinel-lab.service --no-pager
sudo systemctl status redis-sentinel.service --no-pager
```

### 4.2 Check Redis roles and replication

Run the following authenticated command against each data instance. Execute from any VM that can reach the Host-only IPs:

```bash
redis-cli -h 192.168.56.101 -p 7100 --askpass INFO replication
redis-cli -h 192.168.56.102 -p 7100 --askpass INFO replication
redis-cli -h 192.168.56.103 -p 7100 --askpass INFO replication
```

For the original topology used in the recorded test, expected roles were:

- VM1: `role:master`
- VM2: `role:slave`, `master_host:192.168.56.101`, `master_link_status:up`
- VM3: `role:slave`, `master_host:192.168.56.101`, `master_link_status:up`

After a completed failover, the current primary may be VM3 instead. Always identify the live primary from Sentinel rather than assuming VM1 is still primary.

### 4.3 Check Sentinel from VM1

On **VM1**, connect to its local Sentinel:

```bash
redis-cli -h 127.0.0.1 -p 26379 --askpass
```

At the Redis prompt, run:

```redis
PING
SENTINEL CKQUORUM mymaster
SENTINEL MASTER mymaster
SENTINEL SENTINELS mymaster
SENTINEL REPLICAS mymaster
```

Expected checks:

- `PING` returns `PONG`.
- `SENTINEL CKQUORUM mymaster` returns `OK` and reports 3 usable Sentinels.
- `SENTINEL SENTINELS mymaster` lists the other two Sentinel peers without `s_down` or `disconnected` flags.
- `SENTINEL MASTER mymaster` identifies the current primary.
- `SENTINEL REPLICAS mymaster` lists the two replica instances. Healthy replicas should report `master-link-status:ok`.

Exit the Redis prompt with:

```redis
QUIT
```

If quorum is not healthy or a replica link is down, stop here and troubleshoot before simulating a failure.

### 4.4 Create a test key on the current primary

Use the IP returned by `SENTINEL GET-MASTER-ADDR-BY-NAME mymaster` as the current primary IP. In the recorded test, the initial primary was VM1 (192.168.56.101).

Connect to the current primary:

```bash
redis-cli -h <CURRENT_PRIMARY_IP> -p 7100 --askpass
```

At the Redis prompt:

```redis
SET sentinel-failover-test "before-failover"
GET sentinel-failover-test
QUIT
```

Expected: `SET` returns `OK`, and `GET` returns `before-failover`.

Verify the key on **both replicas** by running (replace the IPs only if the live topology differs):

```bash
redis-cli -h 192.168.56.102 -p 7100 --askpass GET sentinel-failover-test
redis-cli -h 192.168.56.103 -p 7100 --askpass GET sentinel-failover-test
```

Both should return `before-failover`. Do not stop the primary until both replicas have the test key.

## 5. Simulate primary failure

### 5.1 Identify the active primary again

On VM1's Sentinel CLI (or another reachable Sentinel), run:

```redis
SENTINEL GET-MASTER-ADDR-BY-NAME mymaster
```

Record the returned IP and port. The IP tells you which VM currently owns the primary role.

### 5.2 Stop only the Redis data service on the current primary VM

Log in to the VM that currently hosts the primary. For example, if Sentinel reports `192.168.56.101:7100`, run this on **VM1**:

```bash
sudo systemctl stop redis-sentinel-lab.service
sudo systemctl is-active redis-sentinel-lab.service
```

Expected service state: `inactive`.

**Do not stop `redis-sentinel.service`.** Sentinel must remain running on all three VMs so it can detect the failed Redis primary and coordinate promotion.

Do not stop Redis Cluster services or power off the VM for this test. We are simulating a Redis data-process failure, not a VM/network failure.

### 5.3 Confirm Sentinel remains active

On the failed-primary VM, run:

```bash
sudo systemctl is-active redis-sentinel.service
```

Expected: `active`.

On the other two VMs, confirm Sentinel is also active:

```bash
sudo systemctl is-active redis-sentinel.service
```

Wait approximately 10–20 seconds. Sentinel detection and promotion may take longer depending on timing and the configured failover process.

## 6. Verify automatic promotion

### 6.1 Query Sentinel

From any VM whose Sentinel is running, connect:

```bash
redis-cli -h 127.0.0.1 -p 26379 --askpass
```

Run:

```redis
SENTINEL CKQUORUM mymaster
SENTINEL MASTER mymaster
SENTINEL GET-MASTER-ADDR-BY-NAME mymaster
SENTINEL REPLICAS mymaster
```

Expected:

- Quorum remains available: `OK 3 usable Sentinels`.
- `SENTINEL MASTER` reports the promoted replica as `master`.
- `SENTINEL GET-MASTER-ADDR-BY-NAME` returns the new primary IP and port `7100`.
- The failed old primary is no longer the active master.

In the recorded test on 20 November 2026, VM3 (192.168.56.103:7100) was promoted. VM2 remained a replica.

### 6.2 Verify the promoted Redis instance

From any VM, run against the IP returned by Sentinel:

```bash
redis-cli -h <NEW_PRIMARY_IP> -p 7100 --askpass INFO replication
```

Expected:

```text
role:master
```

The promoted primary should report its connected replica(s) as online when they have reconnected.

### 6.3 Verify the pre-failover data survived

```bash
redis-cli -h <NEW_PRIMARY_IP> -p 7100 --askpass GET sentinel-failover-test
```

Expected:

```text
before-failover
```

### 6.4 Verify writes after promotion

Connect to the new primary:

```bash
redis-cli -h <NEW_PRIMARY_IP> -p 7100 --askpass
```

Run:

```redis
SET failover-write-test "written-after-failover"
GET failover-write-test
QUIT
```

Expected: `SET` returns `OK`; `GET` returns `written-after-failover`.

Verify the new value on the surviving replica. For example, if VM2 is the replica:

```bash
redis-cli -h 192.168.56.102 -p 7100 --askpass GET failover-write-test
```

Expected: `written-after-failover`.

Do not start the failed primary until the promoted primary and surviving replica have been verified.

## 7. Recover the former primary as a replica

### 7.1 Check the former primary's Redis replication credential

On the **former-primary VM**, inspect whether the Redis data config contains a `masterauth` directive:

```bash
sudo awk '
$1=="masterauth" {print "masterauth: CONFIGURED"; next}
$1=="requirepass" {print "requirepass: CONFIGURED"; next}
$1=="replicaof" {print "replicaof: CONFIGURED"}
' /etc/redis/redis-7100.conf
```

The former primary must be able to authenticate to the new primary. Its `masterauth` value must match the Redis data `requirepass` on the new primary. Compare the values privately on the VMs; do not print or paste either secret.

If `masterauth` is missing or incorrect, edit the former-primary config:

```bash
sudo vi /etc/redis/redis-7100.conf
```

Add or correct this line using the actual Redis **data** password used by the new primary:

```conf
masterauth <REDIS_DATA_PASSWORD>
```

Do not confuse this with the separate Sentinel password. Do not commit the real value.

### 7.2 Start the former primary's Redis data service

On the former-primary VM:

```bash
sudo systemctl start redis-sentinel-lab.service
sudo systemctl status redis-sentinel-lab.service --no-pager
```

Expected: service is `active (running)`.

Leave `redis-sentinel.service` running. Sentinel should detect the returning Redis instance and configure it as a replica of the current primary.

Wait approximately 10–20 seconds before checking. If replication does not recover, inspect the Redis data log:

```bash
sudo tail -n 50 /var/log/redis/redis-7100.log
```

A repeated `NOAUTH Authentication required` or `WRONGPASS` indicates that the replication authentication still needs attention. Do not disable protected mode or remove authentication to work around it.

### 7.3 Verify the recovered node

On the former-primary VM:

```bash
redis-cli -h 127.0.0.1 -p 7100 --askpass INFO replication
```

Expected pattern:

```text
role:slave
master_host:<CURRENT_PRIMARY_IP>
master_port:7100
master_link_status:up
```

The recovered node must point to the promoted primary, not to its old primary role.

### 7.4 Verify the full topology on the current primary

On the current primary VM, run:

```bash
redis-cli -h 127.0.0.1 -p 7100 --askpass INFO replication
```

Expected:

- `role:master`
- `connected_slaves:2`
- Both replica entries show `state=online`.

Also verify the post-failover write from both replicas:

```bash
redis-cli -h 192.168.56.101 -p 7100 --askpass GET failover-write-test
redis-cli -h 192.168.56.102 -p 7100 --askpass GET failover-write-test
```

Use the actual two replica IPs if the roles differ. Both should return the value written on the new primary.

Finally, re-check Sentinel:

```redis
SENTINEL CKQUORUM mymaster
SENTINEL MASTER mymaster
SENTINEL REPLICAS mymaster
```

The current primary should remain master, both replicas should be listed, and each replica should report `master-link-status:ok`.

## 8. Troubleshooting reference

| Symptom | Meaning | What to inspect |
|---|---|---|
| `NOAUTH Authentication required` in a replica log | Replica reached the primary but did not authenticate | Former replica's `masterauth`; current primary's `requirepass` |
| `WRONGPASS invalid username-password pair` | Supplied password does not match the active Redis user, or the user is disabled | Verify the correct Redis data password and active ACL/default-user configuration |
| `master_link_status:down` | Replica is not currently replicating from its master | `/var/log/redis/redis-7100.log`, firewall, master address, `masterauth` |
| `SENTINEL CKQUORUM` returns `NOQUORUM` | Not enough Sentinels are reachable to authorize failover | Sentinel service status, TCP/26379 reachability, peer authentication, Sentinel logs |
| Peer has `s_down` or `disconnected` flags | Sentinel considers a peer unavailable | Check peer Sentinel service, firewall TCP/26379 and Sentinel authentication |
| Old primary returns but remains `master_link_status:down` | It has not authenticated or connected to the new primary | Confirm `masterauth` matches the new primary's Redis data password; inspect Redis log |

Useful diagnostics:

```bash
# Redis data service and Sentinel service
sudo systemctl status redis-sentinel-lab.service --no-pager
sudo systemctl status redis-sentinel.service --no-pager

# Redis data log
sudo tail -n 50 /var/log/redis/redis-7100.log

# Sentinel log
sudo tail -n 50 /var/log/redis/sentinel.log

# Listening ports
sudo ss -lntp | grep -E ':(7100|26379)'

# Firewall rules
sudo firewall-cmd --list-all
```

Do not use `protected-mode no`, disable SELinux, or open Redis ports to unrestricted networks as troubleshooting shortcuts.

## 9. Recorded test result — 20 November 2026

The lab completed the following end-to-end exercise:

1. Initial topology: VM1 was primary; VM2 and VM3 were replicas.
2. Wrote `sentinel-failover-test=before-failover` to VM1 and verified the value on both replicas.
3. Stopped only VM1's `redis-sentinel-lab.service`; VM1's Sentinel remained active.
4. Sentinel quorum remained healthy: `OK 3 usable Sentinels`.
5. Sentinel promoted VM3 (192.168.56.103:7100). `SENTINEL MASTER mymaster` reported VM3 as master.
6. VM3's `INFO replication` reported `role:master`; VM2 was online as a replica.
7. The original test key remained readable on VM3.
8. Wrote a new value on VM3 and verified it on VM2.
9. Restarted VM1's Redis data service. Its first replication attempts failed because VM1 did not have the required `masterauth` setting.
10. Added the correct Redis data `masterauth` to VM1's `/etc/redis/redis-7100.conf` and restarted the Redis data service.
11. VM1 then reported `role:slave`, `master_host:192.168.56.103`, and `master_link_status:up`.
12. Final checks confirmed VM3 as primary with VM1 and VM2 as online replicas, and the post-failover write was readable from the replicas.

Result: automatic failover, continued writes, and former-primary recovery were verified in this lab.

## 10. Final checklist

- [ ] All three Redis data services are active.
- [ ] All three Sentinel services are active.
- [ ] One current primary is reported by Sentinel.
- [ ] Both replicas report `master_link_status:up`.
- [ ] Current primary reports `connected_slaves:2` and both are online.
- [ ] `SENTINEL CKQUORUM mymaster` reports three usable Sentinels.
- [ ] Both test keys can be read from the current primary and replicas.
- [ ] Redis data and Sentinel credentials have been rotated if exposed.
- [ ] No real credentials, password hashes, or private keys are committed to Git.

## 11. Operational notes

- Redis replication is asynchronous. A failover can lose writes that had not reached a replica before the primary failed. This lab verifies the specific test keys used; it does not prove zero data loss under every failure timing.
- The three VMs are separate guests but share the same VirtualBox host. This is not equivalent to three independent physical failure domains.
- The Redis Cluster and Sentinel topologies are separate. This runbook only operates on the standalone Redis data service on port 7100 and Sentinel on port 26379.
