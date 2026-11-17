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

## Step 4 - Create the VirtualBox Host-only network

Created a Host-only network in VirtualBox Network Manager with the following settings:

| Setting | Value |
|---|---|
| Network | VirtualBox Host-Only Ethernet Adapter |
| Host IPv4 address | `192.168.56.1` |
| IPv4 prefix | `/24` |
| DHCP server | Disabled |

The Host-only network provides a private network between the host machine and the VMs. DHCP is disabled because private VM addresses will be assigned manually in a later step.

The NAT adapter will be retained separately to provide internet access to the VMs.

## Step 5 - Attach the Host-only network to each VM

For each VM, open **Settings > Network > Adapter 2** and configure:

- Enable Network Adapter.
- Attached to: **Host-only Adapter**.
- Name: **VirtualBox Host-Only Ethernet Adapter**.

Keep **Adapter 1** attached to **NAT**.

> Complete and verify this step on all three VMs before proceeding.

## Network interface inventory

Record the guest interface names and addresses after booting the VMs with both adapters enabled.

| VM | NAT interface/address | Host-only interface/address |
|---|---|---|
| redis-vm1 | To be recorded | To be recorded |
| redis-vm2 | To be recorded | To be recorded |
| redis-vm3 | To be recorded | To be recorded |

## Completion status

- [x] Install AlmaLinux 9.8 on all three VMs.
- [x] Assign a unique hostname to each VM.
- [x] Inspect network interfaces using `ip -br addr`.
- [x] Create the VirtualBox Host-only network.
- [x] Disable the Host-only network DHCP server.
- [ ] Attach the Host-only network as Adapter 2 on all three VMs.
- [ ] Assign static private IP addresses.
- [ ] Verify VM-to-VM connectivity.

**Next:** Attach the Host-only network to Adapter 2 on each VM, then boot the VMs and identify the guest interface names before assigning static IP addresses.
