# Redis Sentinel High Availability Lab

## 1. Purpose and isolation

Member B uses a separate standalone Redis primary-replica deployment managed by Redis Sentinel. It is independent of Member A's Redis Cluster.

The Sentinel lab uses Redis data port 7100 and Sentinel port 26379. Existing Cluster ports 7000–7005 and Cluster bus ports 17000–17005 must remain untouched.

## 2. VM topology

| VM | Host-only IP | Redis role | Redis port | Planned Sentinel port |
|---|---|---|---:|---:|
| redis-vm1 | 192.168.56.101 | Primary | 7100 | 26379 |
| redis-vm2 | 192.168.56.102 | Replica | 7100 | 26379 |
| redis-vm3 | 192.168.56.103 | Replica | 7100 | 26379 |

Planned Sentinel monitoring name: mymaster; planned quorum: 2 with three Sentinel instances.

## 3. Redis data-instance configuration examples

The following are the intended configuration patterns. The bind address must match the VM. Passwords are placeholders; never commit real credentials.

### VM1 — primary

File: /etc/redis/redis-7100.conf

```conf
bind 127.0.0.1 192.168.56.101
protected-mode yes
port 7100
daemonize no
supervised no
logfile /var/log/redis/redis-7100.log
dir /var/lib/redis/7100
save 3600 1 300 100 60 10000
appendonly yes
appendfsync everysec
requirepass <REDIS_PASSWORD>
```

### VM2 — replica

File: /etc/redis/redis-7100.conf

```conf
bind 127.0.0.1 192.168.56.102
protected-mode yes
port 7100
daemonize no
supervised no
logfile /var/log/redis/redis-7100.log
dir /var/lib/redis/7100
save 3600 1 300 100 60 10000
appendonly yes
appendfsync everysec
replicaof 192.168.56.101 7100
masterauth <REDIS_PASSWORD>
requirepass <REDIS_PASSWORD>
```

### VM3 — replica

File: /etc/redis/redis-7100.conf

```conf
bind 127.0.0.1 192.168.56.103
protected-mode yes
port 7100
daemonize no
supervised no
logfile /var/log/redis/redis-7100.log
dir /var/lib/redis/7100
save 3600 1 300 100 60 10000
appendonly yes
appendfsync everysec
replicaof 192.168.56.101 7100
masterauth <REDIS_PASSWORD>
requirepass <REDIS_PASSWORD>
```

These are documented intended examples based on the instructions issued during setup. VM3's actual file must still be inspected because its replication link is down.

## 4. systemd service

The same service unit was created on all three VMs:

File: /etc/systemd/system/redis-sentinel-lab.service

```ini
[Unit]
Description=Redis Sentinel Lab Instance
After=network.target

[Service]
Type=simple
User=redis
Group=redis
ExecStart=/usr/bin/redis-server /etc/redis/redis-7100.conf
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Apply and start:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now redis-sentinel-lab
sudo systemctl status redis-sentinel-lab
```

Despite its historical name, this unit runs a Redis data server on port 7100. It is not the Sentinel process. Sentinel will have its own configuration and service.

## 5. Supporting preparation

On each VM, the Redis data and log paths were prepared:

```bash
sudo mkdir -p /var/lib/redis/7100
sudo chown redis:redis /var/lib/redis/7100
sudo chmod 750 /var/lib/redis/7100
sudo touch /var/log/redis/redis-7100.log
sudo chown redis:redis /var/log/redis/redis-7100.log
```

SELinux port labeling was checked/updated for TCP 7100 using:

```bash
sudo semanage port -a -t redis_port_t -p tcp 7100
```

On these systems the command reported that TCP/7100 was already defined and was modified. VM1 firewall was confirmed to allow TCP/7100 from 192.168.56.0/24. Existing Cluster firewall rules were retained. vm.overcommit_memory=1 was applied and persisted on VM1, VM2, and VM3.

## 6. Execution and verification log

### Network preflight

- VM-to-VM ping succeeded between the three Host-only addresses.
- From VM2, ping to VM1 succeeded (4/4 packets).
- From VM2, nc -vz 192.168.56.101 7100 connected successfully.
- VM1 was confirmed listening on 192.168.56.101:7100 and loopback.

### VM1 primary

- Redis service redis-sentinel-lab was created, enabled, and started.
- Local Redis PING returned PONG.
- Redis was listening on loopback and VM1's Host-only address.
- Remote access initially failed because protected mode rejected unauthenticated non-loopback connections.
- Authentication was then configured on the primary and replica.

### VM2 replica

- Redis service redis-sentinel-lab was created, enabled, and started.
- Local Redis PING returned PONG.
- Initial replication failed with a protected-mode denial from the primary.
- After matching primary authentication and configuring masterauth, VM2 reported:

```text
role:slave
master_host:192.168.56.101
master_port:7100
master_link_status:up
master_sync_in_progress:0
```

Status: replication verified.

### VM3 replica

- Data directory and log file were prepared.
- Redis config and the same systemd unit were created.
- Service was enabled and reported active/running.
- INFO replication reported role:slave and master 192.168.56.101:7100, but master_link_status:down.
- master_total_sync_attempts was reported as 54036, indicating repeated connection/synchronization attempts.

Status: not yet verified. Do not proceed to Sentinel deployment until VM3 replication is healthy.

Next diagnostics to collect on VM3:

```bash
sudo tail -50 /var/log/redis/redis-7100.log
sudo cat /etc/redis/redis-7100.conf
sudo ss -lntp | grep 7100
redis-cli -h 192.168.56.101 -p 7100 PING
```

For the remote PING, authenticate at the interactive Redis CLI prompt rather than placing the password in shell history or process arguments. Redact all passwords before sharing output.

Compare VM3's bind address, master address, masterauth, and password configuration with the working VM2 pattern. Do not disable SELinux or protected mode as a shortcut.


### VM3 diagnostic update — 20 November 2026

Further log inspection narrowed the failure to Redis authentication during replication:

- VM3 log reports that the primary answers PING, but subsequent REPLCONF and PSYNC/SYNC stages return `-NOAUTH Authentication required`.
- VM3's TCP listeners were confirmed on both `192.168.56.103:7100` and `127.0.0.1:7100`.
- A first attempt to print a sanitized config with sed failed due to an incorrect capture-group placement; no configuration was changed by that command.
- The next step is to inspect the sanitized effective file contents and confirm that `masterauth` matches the primary's configured password. Do not expose the password in terminal output or chat.

Use this safe inspection command on VM3:

```bash
sudo awk '$1=="requirepass" || $1=="masterauth" {$2="<REDACTED>"} {print}' /etc/redis/redis-7100.conf
```


### VM3 password fingerprint comparison — 20 November 2026

- SHA-256 of VM1's configured `requirepass` value was collected without displaying the secret.
- SHA-256 of VM3's configured `masterauth` value was collected the same way.
- Both hashes matched: `8e4073bf...5a074ba` (full hash intentionally not necessary in this narrative).
- This confirms the two on-disk values match. It does not yet prove the running Redis processes loaded the same values or that VM1's live authentication configuration matches its file.

Next: inspect live Redis authentication settings on VM1 and VM3 using authenticated `CONFIG GET` commands, without printing secrets. Then inspect VM3 logs after a controlled service restart only if the live settings indicate it is needed.


### VM3 replication recovered — 20 November 2026

A later VM3 replication check reported:

```text
role:slave
master_host:192.168.56.101
master_port:7100
master_link_status:up
master_sync_in_progress:0
```

This confirms the VM3-to-VM1 replication link was up at the time of the check.

The same command attempted authentication using the literal instructional placeholder rather than the actual secret and returned an AUTH error stating that no password was configured for VM3's default user. Therefore, VM3's local client-authentication state is not yet verified and may be unprotected, even though its replication link is currently up. Do not infer that VM3's local requirepass is active from the on-disk directive alone.

Next: verify unauthenticated local PING behavior on VM3, then securely configure and persist local client authentication if absent. Re-check replication afterward. Do not expose secrets in command output or commit them.


### VM3 ACL password mismatch identified — 20 November 2026

- VM1's configured requirepass value was hashed without a trailing newline: `13a3ec27...1263f6`.
- VM3's explicit `user default` ACL password hash is `c44d5c93...b3d775`.
- These fingerprints differ. This establishes that VM3's explicit default-user ACL password hash does not correspond to the configured VM1 requirepass value.
- The earlier VM1 requirepass versus VM3 masterauth comparison used `sha256sum` with a trailing newline; the newer VM1 hash removed that newline. Those earlier fingerprints are therefore not directly comparable to the current hash.
- Before changing ACLs, compare VM1 requirepass and VM3 masterauth using the same newline-free method. Then align VM3's default-user authentication and masterauth with the primary using a controlled, secure procedure.


### VM3 authentication comparison clarified — 20 November 2026

A consistent comparison was made after stripping quotation marks from the config values and hashing without a trailing newline:

- VM1 `requirepass` fingerprint: `13a3ec27...1263f6`.
- VM3 `masterauth` fingerprint: `13a3ec27...1263f6`.
- Result: the VM3 replication credential matches VM1's configured primary password.

The VM3 file inspection also showed a `requirepass` value that appears to omit the first character compared with `masterauth`, and an explicit `user default` ACL hash that differs from the primary password fingerprint. Therefore, the remaining issue concerns VM3's local default-user authentication configuration, not the on-disk masterauth value. Preserve the matching masterauth value while resolving the default-user ACL/password conflict.


### VM3 local authentication check — 20 November 2026

A fresh unauthenticated local PING to VM3 returned `NOAUTH Authentication required`. This confirms that local client authentication is active in the running instance. The expected credential still failed earlier, consistent with the mismatch between the explicit `user default` ACL hash and the primary password. VM3's on-disk `masterauth` fingerprint matches VM1's `requirepass` fingerprint.

## 7. Security notes

- Do not commit actual Redis passwords, ACL secrets, TLS private keys, or private key material.
- Use placeholders such as <REDIS_PASSWORD> in documentation.
- A credential was pasted into the project chat during setup; treat it as exposed and rotate it before considering the lab complete.
- Authentication is an interim baseline. Planned Member B security work still includes Redis ACLs and TLS.
- Keep TCP/7100 restricted to the private Host-only subnet.

## 8. Remaining work

The following checklist reflects the state after the final verification recorded below.

- [x] Diagnose VM3 replication link and verify master_link_status:up.
- [x] Verify data replication from VM1 to both replicas.
- [x] Deploy one Sentinel process on each VM, using TCP/26379.
- [x] Configure Sentinel monitoring for mymaster with quorum 2.
- [x] Verify Sentinel discovery and peer communication.
- [x] Perform a controlled primary-failure test and verify automatic promotion.
- [x] Verify Sentinel client discovery of the promoted primary.
- [x] Verify post-failover writes and replication to the surviving replica.
- [x] Recover the former primary as a replica and verify its replication link.
- [ ] Rotate credentials exposed during troubleshooting and verify authentication on all nodes.
- [ ] Implement a complete Redis ACL user/permission model.
- [ ] Implement TLS for Redis client, replication, and Sentinel communication as designed.
- [ ] Add sanitized configuration examples and screenshots/evidence.


### Primary and both replicas verified — 20 November 2026

The VM3 authentication mismatch was corrected. The explicit conflicting `user default` ACL line was removed from VM3's configuration, and a newline-free SHA-256 comparison confirmed that VM3's `masterauth` and `requirepass` values match.

After restarting the VM3 Redis data service, authenticated local access succeeded and `INFO replication` reported:

```text
role:slave
master_host:192.168.56.101
master_port:7100
master_link_status:up
master_sync_in_progress:0
```

The final replication check on VM1 reported:

```text
role:master
connected_slaves:2
slave0:ip=192.168.56.102,port=7100,state=online,offset=3402,lag=0
slave1:ip=192.168.56.103,port=7100,state=online,offset=3402,lag=0
master_repl_offset:3402
```

Result: VM1 is the primary; VM2 and VM3 are both connected replicas and were online with zero reported lag at the time of verification. This completes the standalone Redis primary-replica foundation. The earlier VM3 diagnostic entries above are historical; this final verification supersedes their pending/down status.

Security reminder: a Redis credential was pasted into the project conversation. Treat it as compromised and rotate it consistently on all three Redis instances before finalizing the lab. Do not commit the credential or its hashes.


## 9. Sentinel deployment and final verification — 20 November 2026

The Sentinel layer was deployed separately from the Redis data processes.

### Sentinel services and configuration

- Sentinel configuration: `/var/lib/redis/sentinel/sentinel.conf` on each VM.
- Sentinel systemd unit: `/etc/systemd/system/redis-sentinel.service`.
- Sentinel process: `/usr/bin/redis-sentinel /var/lib/redis/sentinel/sentinel.conf`.
- Sentinel port: TCP/26379.
- Monitored master name: `mymaster`.
- Configured quorum: `2`.
- Sentinel log: `/var/log/redis/sentinel.log`.
- The Redis data service remains `redis-sentinel-lab.service` on TCP/7100. Despite the unit's historical name, it runs `redis-server`, not Sentinel.

The Sentinel service was enabled and running on all three VMs. The Sentinel default-user ACL entries containing `nopass` were removed from the three Sentinel configuration files so that the configured `requirepass` is enforced. Local unauthenticated PING checks returned `NOAUTH Authentication required`, and authenticated remote PING checks from VM1 to VM2 and VM3 returned `PONG`.

### Sentinel quorum and peer discovery

From VM1, the following command returned:

```text
SENTINEL CKQUORUM mymaster
OK 3 usable Sentinels. Quorum and failover authorization can be reached
```

`SENTINEL SENTINELS mymaster` listed VM2 (192.168.56.102:26379) and VM3 (192.168.56.103:26379). Neither peer had `s_down` or `disconnected` flags.

`SENTINEL REPLICAS mymaster` showed VM2 and VM3 as replicas of the then-current VM1 primary, each with `master-link-status:ok`.

### Automatic failover test

The controlled test began with VM1 as the Redis primary and VM2/VM3 as replicas.

1. Wrote `sentinel-failover-test=before-failover` on VM1.
2. Read the same key successfully from VM2 and VM3 before stopping the primary.
3. Stopped only `redis-sentinel-lab.service` on VM1. The `redis-sentinel.service` process on VM1 remained active.
4. Sentinel quorum remained available with three usable Sentinels.
5. Sentinel promoted VM3 (192.168.56.103:7100). `SENTINEL MASTER mymaster` reported VM3 with the `master` flag and config epoch 1.
6. VM3's `INFO replication` reported `role:master`; VM2 was connected as an online replica.
7. The pre-failover key remained readable on VM3.
8. A new value written on VM3 was read successfully from VM2.

### Former-primary recovery

When VM1's Redis data service was restarted, Sentinel identified it as a replica of VM3. Its first replication attempts failed because VM1 did not have the required Redis data `masterauth` directive. The Redis log showed `NOAUTH Authentication required` during REPLCONF/PSYNC.

The missing `masterauth` was added to VM1's `/etc/redis/redis-7100.conf`, matching the Redis data `requirepass` used by the new primary. After restarting the VM1 Redis data service, its replication status reported:

```text
role:slave
master_host:192.168.56.103
master_port:7100
master_link_status:up
```

Final verification confirmed VM3 as primary with VM1 and VM2 as online replicas. The post-failover test value was readable from the replicas.

### Final Member B state

| Component | Final state |
|---|---|
| VM1 Redis data process | Replica of VM3; replication link up |
| VM2 Redis data process | Replica of VM3; online |
| VM3 Redis data process | Primary |
| VM1 Sentinel | Running |
| VM2 Sentinel | Running |
| VM3 Sentinel | Running |
| Sentinel quorum | Three usable Sentinels; quorum 2 |
| Automatic failover | Verified; VM1 failure led to VM3 promotion |
| Post-failover write replication | Verified |
| Former-primary recovery | Verified |

The detailed, repeatable procedure with per-VM commands, expected outputs, recovery instructions, and troubleshooting is in [Redis Sentinel Failover and Recovery Runbook](07-redis-sentinel-failover-testing.md). The files, sanitized configuration contents, systemd units, directories, permissions, firewall rules, and build sequence are documented in [Redis Sentinel Configuration and File Reference](08-redis-sentinel-configuration.md).

### Security and scope notes

- A Redis data credential and a Sentinel credential were exposed during interactive troubleshooting. They must be rotated before the lab is considered security-complete. Do not commit real credentials, hashes, or secret values.
- The Sentinel authentication fix and Redis replication authentication fix are separate: Sentinel client authentication uses the Sentinel configuration, while Redis replica authentication to the data primary uses `masterauth` in the Redis data configuration.
- TLS for Redis client, replication, and Sentinel traffic remains future work; authentication alone is not TLS encryption.
- This Sentinel deployment manages only the standalone Redis service on port 7100. It does not manage or alter Member A's Redis Cluster on ports 7000–7005.
