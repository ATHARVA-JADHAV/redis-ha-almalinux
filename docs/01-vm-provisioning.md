# 01 - VM Provisioning

## Objective

Prepare three AlmaLinux 9.8 virtual machines that will host the Redis Cluster and Redis Sentinel lab. The same three VMs will be reused for both deployment modes; the two Redis architectures will be configured and tested separately.

## Virtual machine inventory

| VM | Hostname | Operating system | Host-only IPv4 address |
|---|---|---|---|
| VM 1 | `redis-vm1` | AlmaLinux 9.8 | `192.168.56.101/24` |
| VM 2 | `redis-vm2` | AlmaLinux 9.8 | `192.168.56.102/24` |
| VM 3 | `redis-vm3` | AlmaLinux 9.8 | `192.168.56.103/24` |

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

The Host-only network provides a private network between the host machine and the VMs. DHCP is disabled because private VM addresses are assigned manually.

## Step 5 - Configure two network adapters per VM

Configured the following network adapters on all three VMs:

| Adapter | VirtualBox attachment | Purpose |
|---|---|---|
| Adapter 1 | NAT | Outbound internet access for package installation and system updates |
| Adapter 2 | Host-only Adapter | Private communication between the host and the three VMs |

For Adapter 2, selected **VirtualBox Host-Only Ethernet Adapter**.

The NAT adapter provides outbound connectivity. The Host-only adapter creates a separate private network so that the Redis nodes can communicate using stable private addresses.

## Step 6 - Assign static IPv4 addresses

Configured static IPv4 addresses on the Host-only interfaces using NetworkManager (`nmcli`).

| VM | Host-only IPv4 address |
|---|---|
| redis-vm1 | `192.168.56.101/24` |
| redis-vm2 | `192.168.56.102/24` |
| redis-vm3 | `192.168.56.103/24` |

The `/24` prefix corresponds to subnet mask `255.255.255.0`. No default gateway is configured on the Host-only interfaces; the NAT interface remains responsible for the default route and internet access.

## Step 7 - Verify VM-to-VM connectivity

Tested connectivity between the three VMs using `ping`. Successful ping replies were received between the VMs, confirming basic IP connectivity over the private Host-only network.

Example test from redis-vm1:

```bash
ping -c 4 192.168.56.102
ping -c 4 192.168.56.103
```

Equivalent tests were performed from the other VMs to verify communication in both directions.

## Step 8 - Update the operating systems

Updated all three AlmaLinux systems successfully after resolving the NAT connectivity issue on VM3.

```bash
sudo dnf update -y
```

The `dnf update` operation refreshed installed packages using the configured repositories. The NAT adapter provides the outbound connectivity required to reach those repositories; the Host-only interface remains dedicated to private lab communication.

**Result:** The system update completed successfully on redis-vm1, redis-vm2, and redis-vm3.

## Step 9 - Verify internet access through NAT

Verified that all three VMs had working outbound connectivity through their NAT adapters as part of the successful system update process.

The NAT interface is used for package downloads and operating-system updates. Redis node-to-node communication will use the Host-only network and its static addresses.

## Completion status

- [x] Install AlmaLinux 9.8 on all three VMs.
- [x] Assign a unique hostname to each VM.
- [x] Inspect network interfaces using `ip -br addr`.
- [x] Create the VirtualBox Host-only network.
- [x] Disable the Host-only network DHCP server.
- [x] Configure Adapter 1 as NAT on all three VMs.
- [x] Configure Adapter 2 as Host-only on all three VMs.
- [x] Assign static private IP addresses.
- [x] Verify VM-to-VM connectivity with successful ping tests.
- [x] Resolve the VM3 NAT DHCP issue.
- [x] Update all three AlmaLinux systems successfully.
- [x] Confirm working outbound connectivity required for package updates.

**Next:** Install and verify Redis on all three VMs, then prepare the six independent Redis processes required for the Cluster.
