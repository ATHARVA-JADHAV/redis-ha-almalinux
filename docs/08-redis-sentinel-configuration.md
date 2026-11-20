# Redis Sentinel Configuration and File Reference

## 1. Purpose

This document is the configuration companion to:

- [Redis Sentinel High Availability Lab](06-redis-sentinel-ha.md)
- [Redis Sentinel Failover and Recovery Runbook](07-redis-sentinel-failover-testing.md)

It records the important files created for Member B, what each file does, when it is introduced in the build, the relevant contents, ownership/permissions, and the commands used to create or inspect it.

This is a sanitized reference. Passwords are always represented by placeholders. The live Sentinel configuration is maintained and rewritten by Sentinel at runtime, so treat the examples as templates and compare them with the live files before replacing anything.

## 2. Topology and ports

| VM | Host-only IP | Redis data port | Sentinel port |
|---|---|---:|---:|
| redis-vm1 | 192.168.56.101 | 7100 | 26379 |
| redis-vm2 | 192.168.56.102 | 7100 | 26379 |
| redis-vm3 | 192.168.56.103 | 7100 | 26379 |

- Redis data service: `redis-sentinel-lab.service` (port 7100).
- Sentinel service: `redis-sentinel.service` (port 26379).
- Sentinel monitored master name: `mymaster`.
- Sentinel quorum: `2`.
- Redis Cluster (7000–7005 and 17000–17005) is separate and must not be changed by this guide.

## 3. File inventory and when each file was created

| Path | Created/used during | Purpose |
|---|---|---|
| `/etc/redis/redis-7100.conf` | Redis data-instance setup, before starting the data service | Redis server settings, bind/port, persistence, authentication and initial replication configuration |
| `/etc/systemd/system/redis-sentinel-lab.service` | Redis data-instance setup | Starts `redis-server` with `redis-7100.conf`; despite the unit name, this is the data server |
| `/var/lib/redis/7100/` | Redis data-instance setup | Working/data directory for the port-7100 Redis process |
| `/var/log/redis/redis-7100.log` | Redis data-instance setup | Redis data-server log |
| `/var/lib/redis/sentinel/sentinel.conf` | Sentinel deployment, after the data primary/replicas were healthy | Sentinel-specific configuration and runtime-discovered topology |
| `/etc/systemd/system/redis-sentinel.service` | Sentinel deployment | Starts the Sentinel process with its own configuration |
| `/var/lib/redis/sentinel/` | Sentinel deployment | Sentinel working directory |
| `/var/log/redis/sentinel.log` | Sentinel deployment | Sentinel log |
| `/var/lib/redis/sentinel/sentinel.conf.bak` | Authentication troubleshooting | Backup created before removing the conflicting `user default ... nopass` line on each VM |

The two systemd services are deliberately separate. Starting or stopping the Redis data service does not mean Sentinel is stopped.

## 4. Stage A — Redis data-instance files

### 4.1 Create the data directory and log file

Run on **VM1, VM2, and VM3**:

```bash
sudo mkdir -p /var/lib/redis/7100
sudo chown redis:redis /var/lib/redis/7100
sudo chmod 750 /var/lib/redis/7100

sudo touch /var/log/redis/redis-7100.log
sudo chown redis:redis /var/log/redis/redis-7100.log
```

The Redis process runs as the `redis` user and needs access to its data directory and log file.

### 4.2 Redis data configuration

File on all three VMs:

```text
/etc/redis/redis-7100.conf
```

The common configuration pattern used for the standalone data instances is:

```conf
bind 127.0.0.1 <THIS_VM_HOST_ONLY_IP>
protected-mode yes
port 7100
daemonize no
supervised no
logfile /var/log/redis/redis-7100.log
dir /var/lib/redis/7100
save 3600 1 300 100 60 10000
appendonly yes
appendfsync everysec
requirepass <REDIS_DATA_PASSWORD>
```

Replace `<THIS_VM_HOST_ONLY_IP>` with the address belonging to the VM being configured:

- VM1: `192.168.56.101`
- VM2: `192.168.56.102`
- VM3: `192.168.56.103`

On replicas, the initial replication settings also include:

```conf
replicaof 192.168.56.101 7100
masterauth <REDIS_DATA_PASSWORD>
```

The initial setup used VM1 as primary and VM2/VM3 as replicas. The same Redis data password was used for the primary's `requirepass` and the replicas' `masterauth` so replicas could authenticate to the primary.

**Important after Sentinel failover:** the live primary changed from VM1 to VM3. Sentinel reconfigured the Redis roles at runtime. Do not assume that the original `replicaof` line in a static example represents the current live topology. Check the actual role with `INFO replication` and the current primary with `SENTINEL GET-MASTER-ADDR-BY-NAME mymaster`.

### 4.3 How the data configuration was edited and verified

The configuration file was created/edited on each VM using a privileged editor, then the Redis data service was started. For future edits:

```bash
sudo vi /etc/redis/redis-7100.conf
```

Inspect relevant directives without displaying secret values:

```bash
sudo awk '
$1=="bind" {print}
$1=="port" {print}
$1=="protected-mode" {print}
$1=="replicaof" {print}
$1=="requirepass" {print "requirepass: CONFIGURED"; next}
$1=="masterauth" {print "masterauth: CONFIGURED"; next}
' /etc/redis/redis-7100.conf
```

Never use `cat` or an unfiltered `grep` to paste password-bearing configuration into chat or documentation.

### 4.4 Redis data systemd unit

File on all three VMs:

```text
/etc/systemd/system/redis-sentinel-lab.service
```

Contents:

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

This unit was created for the Redis **data server**, not for Sentinel. The name is historical and potentially confusing; preserve it when following this lab unless you deliberately plan a coordinated service rename.

After creating or changing the unit:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now redis-sentinel-lab.service
sudo systemctl status redis-sentinel-lab.service --no-pager
```

Useful checks:

```bash
sudo systemctl is-active redis-sentinel-lab.service
sudo ss -lntp | grep ':7100'
```

## 5. Stage B — Sentinel directories, log and configuration

This stage was performed after the standalone Redis primary and both replicas were established.

### 5.1 Create Sentinel working directory and log

Run on **VM1, VM2, and VM3**:

```bash
sudo mkdir -p /var/lib/redis/sentinel
sudo chown redis:redis /var/lib/redis/sentinel
sudo chmod 750 /var/lib/redis/sentinel

sudo touch /var/log/redis/sentinel.log
sudo chown redis:redis /var/log/redis/sentinel.log
```

The Sentinel process runs as the `redis` user. The working directory and log must be writable by that user.

### 5.2 Sentinel configuration file

File on all three VMs:

```text
/var/lib/redis/sentinel/sentinel.conf
```

The VM-specific template used for this lab is:

```conf
bind 127.0.0.1 <THIS_VM_HOST_ONLY_IP>
port 26379
protected-mode yes
daemonize no
supervised no
logfile /var/log/redis/sentinel.log
dir /var/lib/redis/sentinel

requirepass <SENTINEL_PASSWORD>

sentinel monitor mymaster 192.168.56.101 7100 2
sentinel auth-user mymaster default
sentinel auth-pass mymaster <REDIS_DATA_PASSWORD>
sentinel down-after-milliseconds mymaster 5000
sentinel failover-timeout mymaster 60000
sentinel parallel-syncs mymaster 1
```

Use the correct VM address for `<THIS_VM_HOST_ONLY_IP>`:

- VM1: `192.168.56.101`
- VM2: `192.168.56.102`
- VM3: `192.168.56.103`

The directives have distinct purposes:

| Directive | Purpose |
|---|---|
| `bind` | Listen on loopback and this VM's Host-only address |
| `port 26379` | Sentinel TCP port |
| `protected-mode yes` | Keep protected mode enabled |
| `requirepass` | Require a password for clients connecting to this Sentinel |
| `sentinel monitor mymaster ... 2` | Monitor the Redis data primary under name `mymaster`; quorum is 2 |
| `sentinel auth-user` / `sentinel auth-pass` | Credentials Sentinel uses when connecting to the monitored Redis data instance |
| `down-after-milliseconds 5000` | Failure-detection interval used by this lab |
| `failover-timeout 60000` | Failover timeout setting |
| `parallel-syncs 1` | Number of replicas configured to re-synchronize in parallel during failover |

The Sentinel password and Redis data password are separate logical credentials. The `sentinel auth-pass` value is the Redis data password for the monitored Redis server; it is not the Sentinel client's own `requirepass`.

**Runtime-managed file:** Sentinel can rewrite its configuration as it discovers peers and changes the monitored primary. After Sentinel has started, the file may contain additional generated `sentinel known-sentinel`, `sentinel known-replica`, configuration epoch, or related runtime lines. Do not repeatedly overwrite a live Sentinel configuration with the short template above. Back it up and inspect the current file before making changes.

### 5.3 Create or edit the Sentinel configuration

The file was prepared on each VM before starting the Sentinel systemd service. For a fresh lab, create/edit it with:

```bash
sudo vi /var/lib/redis/sentinel/sentinel.conf
```

Set ownership and permissions:

```bash
sudo chown redis:redis /var/lib/redis/sentinel/sentinel.conf
sudo chmod 640 /var/lib/redis/sentinel/sentinel.conf
```

The Sentinel process must be able to rewrite its configuration during discovery and failover, so the file must remain writable by the `redis` user. Verify:

```bash
sudo stat -c '%U:%G %a %n' /var/lib/redis/sentinel/sentinel.conf
```

Expected ownership/mode used in this lab: `redis:redis 640`.

### 5.4 Sentinel systemd unit

File on all three VMs:

```text
/etc/systemd/system/redis-sentinel.service
```

Contents:

```ini
[Unit]
Description=Redis Sentinel
After=network.target

[Service]
Type=simple
User=redis
Group=redis
ExecStart=/usr/bin/redis-sentinel /var/lib/redis/sentinel/sentinel.conf
Restart=always
RestartSec=3
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target
```

Create/edit the unit:

```bash
sudo vi /etc/systemd/system/redis-sentinel.service
```

Load, enable and start it:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now redis-sentinel.service
sudo systemctl status redis-sentinel.service --no-pager
```

Verify that it is listening:

```bash
sudo systemctl is-active redis-sentinel.service
sudo ss -lntp | grep ':26379'
```

Expected: Sentinel is active and listens on loopback plus the VM's Host-only address.

## 6. Authentication troubleshooting that occurred in this lab

During troubleshooting, the Sentinel config on all three VMs contained an explicit default-user ACL line with `nopass`, while `requirepass` was also configured. The explicit passwordless ACL entry conflicted with the intended default-user password behavior.

The fix performed on each VM was:

1. Back up the current Sentinel configuration.
2. Remove the explicit `user default ...` line containing `nopass`.
3. Keep `requirepass` configured.
4. Restart Sentinel.
5. Confirm that unauthenticated local PING returns `NOAUTH Authentication required`.
6. Confirm that an authenticated remote PING returns `PONG`.

Commands used on each VM:

```bash
sudo cp /var/lib/redis/sentinel/sentinel.conf /var/lib/redis/sentinel/sentinel.conf.bak
sudo sed -i '/^user default /d' /var/lib/redis/sentinel/sentinel.conf
sudo systemctl restart redis-sentinel.service
```

Verify without printing the password:

```bash
sudo grep -E '^(requirepass|user default|protected-mode)' /var/lib/redis/sentinel/sentinel.conf \
  | sed 's/^requirepass .*/requirepass [CONFIGURED]/'
```

Local authentication check:

```bash
redis-cli -h 127.0.0.1 -p 26379 PING
```

Expected without a password: `NOAUTH Authentication required`.

Authenticated remote test from VM1:

```bash
redis-cli --askpass -h 192.168.56.102 -p 26379 PING
redis-cli --askpass -h 192.168.56.103 -p 26379 PING
```

Expected: `PONG` for both.

Do not remove authentication, disable protected mode, or paste secrets into terminal output to resolve an authentication error. A password exposed during troubleshooting must be rotated.

## 7. Redis replica authentication during failover recovery

Sentinel authentication and Redis replication authentication are different.

When VM1 was restarted after VM3 had been promoted, VM1 correctly appeared as a replica of VM3 but initially showed `master_link_status:down`. Its Redis log showed `NOAUTH Authentication required` during replication negotiation.

The missing setting was `masterauth` in VM1's Redis data config:

```conf
masterauth <REDIS_DATA_PASSWORD>
```

The value must match the current primary's Redis data `requirepass`. After adding it and restarting only the VM1 Redis data service, VM1 reported:

```text
role:slave
master_host:192.168.56.103
master_port:7100
master_link_status:up
```

To inspect whether the directives exist without disclosing their values:

```bash
sudo awk '
$1=="requirepass" {print "requirepass: CONFIGURED"; next}
$1=="masterauth" {print "masterauth: CONFIGURED"; next}
$1=="replicaof" {print "replicaof: CONFIGURED"}
' /etc/redis/redis-7100.conf
```

Never paste the actual output of an unfiltered `grep requirepass` or `grep masterauth` into a shared document.

## 8. Firewall and network rules

The lab uses the Host-only subnet `192.168.56.0/24`. TCP/7100 is the Redis data port and TCP/26379 is the Sentinel port.

The following source-restricted rich rules were added where needed on VM1, VM2, and VM3:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.0/24" port port="7100" protocol="tcp" accept'
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="192.168.56.0/24" port port="26379" protocol="tcp" accept'
sudo firewall-cmd --reload
```

Verify:

```bash
sudo firewall-cmd --list-all
```

Keep the rules restricted to the Host-only subnet. Do not expose either port to unrestricted networks.

## 9. SELinux and kernel setting notes

SELinux remained enabled in enforcing mode during the lab. TCP/7100 was checked against the Redis SELinux port type (`redis_port_t`) and the existing port definition was modified/confirmed as needed. Do not disable SELinux to make Redis or Sentinel work.

The kernel setting `vm.overcommit_memory=1` was applied and persisted on VM1, VM2, and VM3. Verify with:

```bash
sysctl vm.overcommit_memory
```

Expected:

```text
vm.overcommit_memory = 1
```

## 10. Useful file and service inspection commands

Run on the relevant VM:

```bash
# Redis data configuration, with secret values redacted
sudo awk '$1=="requirepass" || $1=="masterauth" {$2="<REDACTED>"} {print}' /etc/redis/redis-7100.conf

# Sentinel configuration, with secret values redacted
sudo awk '$1=="requirepass" || $1=="sentinel" && $2=="auth-pass" {$NF="<REDACTED>"} {print}' /var/lib/redis/sentinel/sentinel.conf

# Unit definitions
sudo systemctl cat redis-sentinel-lab.service
sudo systemctl cat redis-sentinel.service

# Service states
sudo systemctl status redis-sentinel-lab.service --no-pager
sudo systemctl status redis-sentinel.service --no-pager

# Logs
sudo tail -n 50 /var/log/redis/redis-7100.log
sudo tail -n 50 /var/log/redis/sentinel.log

# Listening sockets
sudo ss -lntp | grep -E ':(7100|26379)'
```

## 11. Backups and change discipline

Before editing an existing configuration:

```bash
sudo cp /path/to/config /path/to/config.bak
```

After editing a systemd unit:

```bash
sudo systemctl daemon-reload
```

Restart only the service whose configuration was changed, and verify its status and logs before moving to the next VM.

Sentinel can rewrite `sentinel.conf` during normal operation. Take a backup before manual edits and avoid replacing the runtime file with an old template.

## 12. Security checklist

- [ ] Rotate the Redis data credential that was exposed during troubleshooting.
- [ ] Rotate the Sentinel credential that was exposed during troubleshooting.
- [ ] Verify the new Redis data credential consistently on all three VMs.
- [ ] Verify Sentinel authentication on all three VMs and peer discovery after rotation.
- [ ] Verify replica `masterauth` matches the current primary's `requirepass`.
- [ ] Keep passwords, hashes, ACL secrets, and private keys out of Git.
- [ ] Keep SELinux enforcing and firewall rules restricted to the Host-only subnet.
- [ ] Treat TLS as separate remaining work; password authentication does not encrypt network traffic.


## 12. Redis ACL implementation and persistence — 20 November 2026

Redis ACLs were configured for the Member B data instances on TCP/7100 using an external ACL file on all three VMs.

### ACL file and Redis configuration

- ACL file on each VM: `/var/lib/redis/7100/users.acl`.
- Redis data config on each VM includes: `aclfile /var/lib/redis/7100/users.acl`.
- ACL file ownership/mode: `redis:redis`, `640`; the parent data directory remains owned by `redis:redis` with mode `750`.
- The inline default-user ACL entry was removed from the Redis data config when switching to the external ACL file.
- VM1, VM2 and VM3 each have the `default` and `app-user` entries persisted in the external ACL file. Do not copy ACL hashes or passwords into this repository.

### Application user permissions

The `app-user` ACL is restricted to the `app:*` key pattern and has only the following commands enabled:

```text
GET SET DEL EXISTS EXPIRE TTL PERSIST INCR
```

The user has no channel permissions (Pub/Sub is not enabled) and all other commands are denied. The default user remains enabled with the lab's existing permissions.

### Verification performed

- VM1 and VM2: application-user access to an `app:*` key was accepted; access to a key outside that pattern returned `NOPERM`.
- VM3: Redis data service `redis-sentinel-lab.service` was restarted after configuring the external ACL file. The service returned to `active (running)`; `ACL LIST` showed both `default` and `app-user` loaded from the file; `ROLE` confirmed VM3 rejoined as a connected replica of VM1.
- VM1: as `app-user`, `SET app:acl-check persisted` returned `OK`, and `GET app:acl-check` returned the stored value. `GET other:acl-check` returned `NOPERM`.
- VM3: an attempted write as `app-user` returned `READONLY` because VM3 was a replica at the time; this is expected Redis replica behavior, not an ACL denial. Its out-of-pattern read returned `NOPERM`.

The ACL persistence and permission checks are therefore verified. The VM3 replica write test is not a valid write-permission test; the successful VM1 primary write test supplies that verification.

### Security follow-up

Credentials pasted into the project conversation during troubleshooting must be rotated before security sign-off. Keep real passwords and hashes out of Git. TLS remains separate outstanding work; ACL authentication does not encrypt network traffic.


## 13. TLS deployment and final encrypted-port configuration — 20 November 2026

TLS was implemented for Member B's standalone Redis/Sentinel deployment on all three VMs. Member A's Redis Cluster ports (7000–7005 and 17000–17005) were not changed.

### Certificate layout

A private lab CA was generated on VM1. It signed one server certificate per VM, each with its Host-only IP and hostname in the Subject Alternative Name (SAN).

| VM | IP SAN | DNS SAN | Certificate/key |
|---|---|---|---|
| VM1 | 192.168.56.101 | redis-vm1 | /etc/redis/tls/server.crt and server.key |
| VM2 | 192.168.56.102 | redis-vm2 | /etc/redis/tls/server.crt and server.key |
| VM3 | 192.168.56.103 | redis-vm3 | /etc/redis/tls/server.crt and server.key |

The CA certificate is installed at `/etc/redis/tls/ca.crt` on all VMs. The CA private key was used to sign the node certificates and must not be copied into the repository or distributed to the other VMs. Certificate verification with `openssl verify -CAfile ca/ca.crt` returned `OK` for all three node certificates, and each SAN matched its VM IP/hostname.

TLS files were installed under `/etc/redis/tls/`, owned by `redis:redis`; directory mode 750 and file mode 640. No private key, password, or certificate hash is stored in this repository.

### Final ports

| Service | Plaintext port | TLS port | Final state |
|---|---:|---:|---|
| Redis data/client/replication | 7100 | 7101 | Plaintext disabled; TLS active |
| Redis Sentinel | 26379 | 26380 | Plaintext disabled; TLS active |

### Redis data configuration

File: `/etc/redis/redis-7100.conf` on each VM. Sanitized TLS directives:

```conf
port 0
tls-port 7101
tls-cert-file /etc/redis/tls/server.crt
tls-key-file /etc/redis/tls/server.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients no
tls-replication yes
```

The existing bind, persistence, password, ACL-file, and data-directory settings remain in place. `tls-auth-clients no` means clients do not present client certificates; the server certificate is CA-validated and Redis ACL/password authentication remains enabled. This is TLS encryption with server-certificate validation, not mutual TLS.

### Sentinel configuration

File: `/var/lib/redis/sentinel/sentinel.conf` on each VM. The runtime-managed file was updated rather than replaced with a static template. The effective TLS settings are:

```conf
port 0
tls-port 26380
tls-cert-file /etc/redis/tls/server.crt
tls-key-file /etc/redis/tls/server.key
tls-ca-cert-file /etc/redis/tls/ca.crt
tls-auth-clients no
tls-replication yes
```

Sentinel monitors the Redis primary on TLS port 7101. Its discovered replica entries use port 7101 and Sentinel peer entries use port 26380. Sentinel configuration remains runtime-managed; inspect the live file before future edits.

### SELinux and firewall

SELinux remained enforcing. Redis TLS ports 7101 and 26380 were assigned the `redis_port_t` TCP port type on the VMs. TCP/7101 and TCP/26380 were allowed through firewalld for the Host-only lab network. The old plaintext ports 7100 and 26379 were not enabled in the firewall's direct port list; final listener checks confirmed that neither plaintext port had a listening socket.

### TLS verification and failover evidence

- All three Redis TLS listeners on TCP/7101 accepted `redis-cli --tls --cacert /etc/redis/tls/ca.crt ... PING`; each returned `PONG`.
- Sentinel TLS on TCP/26380 returned: `OK 3 usable Sentinels. Quorum and failover authorization can be reached`.
- Redis `ROLE` checks over TLS showed the failover result: VM3 (192.168.56.103) was promoted to primary; VM1 (192.168.56.101) and VM2 (192.168.56.102) were replicas connected to VM3 on port 7101.
- After plaintext shutdown, `ss -lntp` on VM1, VM2 and VM3 showed listeners on 7101 and 26380 only; no listeners on 7100 or 26379.

Useful demonstration commands:

```bash
# Redis TLS client test (replace IP for each VM)
redis-cli --tls --cacert /etc/redis/tls/ca.crt -h <VM_IP> -p 7101 --user default --askpass PING

# Sentinel TLS quorum
redis-cli --tls --cacert /etc/redis/tls/ca.crt -h 192.168.56.101 -p 26380 --user default --askpass SENTINEL CKQUORUM mymaster

# Confirm encrypted listeners and absence of plaintext listeners
ss -lntp | grep -E '7100|7101|26379|26380'
```

### Remaining security follow-up

- [ ] Rotate Redis data and Sentinel credentials that were exposed during troubleshooting.
- [ ] If mutual TLS is a requirement, issue client certificates and deliberately enable client-certificate authentication; the current lab uses `tls-auth-clients no`.
- [ ] Keep the CA private key offline/secured after issuing the certificates.
