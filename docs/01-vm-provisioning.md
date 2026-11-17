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

## Step 5 - Configure two network adapters per VM

Configured the following network adapters on all three VMs:

| Adapter | VirtualBox attachment | Purpose |
|---|---|---|
| Adapter 1 | NAT | Outbound internet access for package installation and system updates |
| Adapter 2 | Host-only Adapter | Private communication between the host and the three VMs |

For Adapter 2, selected **VirtualBox Host-Only Ethernet Adapter**.

The NAT adapter provides outbound connectivity. The Host-only adapter creates a separate private network so that the Redis nodes can communicate with predictable private addresses without relying on ordinary NAT networking for VM-to-VM reachability.

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
- [x] Configure Adapter 1 as NAT on all three VMs.
- [x] Configure Adapter 2 as Host-only on all three VMs.
- [ ] Identify the guest interface names.
- [ ] Assign static private IP addresses.
- [ ] Verify VM-to-VM connectivity.

**Next:** Boot all three VMs and run `ip -br addr` to identify the NAT and Host-only interfaces inside AlmaLinux. Then assign static IP addresses to the Host-only interfaces and test connectivity.
