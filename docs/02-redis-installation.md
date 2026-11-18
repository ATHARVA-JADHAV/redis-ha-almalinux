# 02 - Redis Installation

## Objective

Install Redis Open Source on all three AlmaLinux 9.8 virtual machines using the official Redis RPM repository. At this stage, Redis is installed as software; the six independent server instances required for Redis Cluster will be configured in a later stage.

## Step 1 - Configure the official Redis RPM repository

Created `/etc/yum.repos.d/redis.repo` on each VM:

```ini
[Redis]
name=Redis
baseurl=http://packages.redis.io/rpm/rockylinux9
enabled=1
gpgcheck=1
```

The Redis RPM repository publishes packages for Rocky Linux 9 and AlmaLinux 9. The repository definition makes the Redis packages available through DNF.

## Step 2 - Import the Redis package-signing key

Downloaded and imported the Redis signing key on each VM:

```bash
curl -fsSL https://packages.redis.io/gpg -o /tmp/redis.key
sudo rpm --import /tmp/redis.key
```

RPM signature verification helps confirm that installed packages were signed by a key trusted by the local package manager.

## Step 3 - Install Redis

Installed the Redis package on each VM:

```bash
sudo dnf install redis -y
```

This installs the Redis server and command-line utilities supplied by the package.

## Step 4 - Verify the installed binaries

Checked the Redis server and CLI versions:

```bash
redis-server --version
redis-cli --version
```

These commands confirm that the Redis server executable and command-line client are available. The exact version strings should be recorded from the output on each VM; they are intentionally not hardcoded in this guide.

## Installation result

Redis was installed on all three VMs:

| VM | Host-only address | Redis package |
|---|---|---|
| redis-vm1 | 192.168.56.101 | Installed |
| redis-vm2 | 192.168.56.102 | Installed |
| redis-vm3 | 192.168.56.103 | Installed |

## Important service note

The official Redis RPM installation instructions state that Redis does not start automatically after installation. We are deliberately not enabling or starting the default standalone service yet.

The lab requires six independent Redis processes, each with its own port, configuration, data directory, log, and generated cluster node configuration file. Starting the packaged default service before reviewing its service unit and configuration could occupy the default port and complicate the multi-instance setup.

Before creating the six instances, inspect the package and service layout on each VM. Do not assume paths or service-unit names; verify them from the installed package.

Useful inspection commands:

```bash
rpm -ql redis
systemctl list-unit-files | grep -i redis
systemctl status redis
```

A service reported as inactive is expected if it has not been started. Do not enable or start it as a standalone service during this stage.

## Completion status

- [x] Configure the official Redis RPM repository on all three VMs.
- [x] Import the Redis package-signing key on all three VMs.
- [x] Install the Redis package on all three VMs.
- [x] Verify that `redis-server` and `redis-cli` are available.
- [ ] Record and compare the exact Redis version reported by each VM.
- [ ] Inspect package file paths and the packaged systemd service before multi-instance configuration.

**Next:** Inspect the installed package layout, then prepare six independent Redis instances across the three VMs.
