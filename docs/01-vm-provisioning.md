# 01 - VM Provisioning

## Objective

Prepare three AlmaLinux 9.8 virtual machines that will host the Redis Cluster and Redis Sentinel lab. The same three VMs will be reused for both deployment modes; the two Redis architectures will be configured and tested separately.

## Virtual machine inventory

| VM | Hostname | Operating system |
|---|---|---|
| VM 1 | `redis-vm1` | AlmaLinux 9.8 |
| VM 2 | `redis-vm2` | AlmaLinux 9.8 |
| VM 3 | `redis-vm3` | AlmaLinux 9.8 |

## Step 1 - Install AlmaLinux

Installed AlmaLinux 9.8 on all three VirtualBox virtual machines.

## Step 2 - Configure hostnames

A hostname provides each VM with a distinct name on the network. This makes the machines easier to identify and manage.

Run the corresponding command as root on each VM:

**VM 1**
```bash
hostnamectl set-hostname redis-vm1
```

**VM 2**
```bash
hostnamectl set-hostname redis-vm2
```

**VM 3**
```bash
hostnamectl set-hostname redis-vm3
```

Verify the hostname:

```bash
hostnamectl
```

## Step 3 - Inspect network interfaces

Inspected the network interfaces and their assigned addresses on each VM:

```bash
ip -br addr
```

The `-br` option displays the output in brief format, with one line per interface. The `addr` subcommand displays the IP addresses assigned to those interfaces.

Record the interface names and addresses here after the network configuration is finalized.

| VM | Interface | Address | Status |
|---|---|---|---|
| redis-vm1 | To be recorded | To be configured/confirmed | Pending |
| redis-vm2 | To be recorded | To be configured/confirmed | Pending |
| redis-vm3 | To be recorded | To be configured/confirmed | Pending |

## Completion status

- [x] Install AlmaLinux 9.8 on all three VMs.
- [x] Assign a unique hostname to each VM.
- [x] Inspect network interfaces using `ip -br addr`.
- [ ] Configure the VirtualBox network adapters for the lab.
- [ ] Assign or verify static private IP addresses.
- [ ] Verify VM-to-VM connectivity.

**Next:** Configure the VirtualBox network adapters and establish a private network between the three VMs.