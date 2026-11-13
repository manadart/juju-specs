# Azure Dual-Stack Findings

Date: March 11, 2026

## Summary

The Azure provider currently provisions Juju-managed controller networking as IPv4-only.
This is not limited to public IP allocation. The provider also creates IPv4-only VNet/subnet
address space, attaches only IPv4 NIC IP configurations, and filters IPv6 out of subnet
discovery.

As a result, changing only public IP allocation is not enough to give a controller both IPv4
and IPv6 public addresses. Azure requires an IPv6 public IP to be attached to an IPv6 NIC IP
configuration, which in turn requires IPv6 address space on the target subnet.

## Where Juju Enforces IPv4

### 1. Juju-created network resources are IPv4-only

The Juju-managed VNet and subnets are hard-coded to IPv4 CIDRs:

- `internal/provider/azure/networking.go`
- `internalSubnetPrefix = "192.168.0.0/20"`
- `controllerSubnetPrefix = "192.168.16.0/20"`

The shared network template creates:

- A VNet with only those IPv4 `AddressSpace` prefixes
- Subnets with only those IPv4 `AddressPrefix` values

Relevant code:

- [internal/provider/azure/networking.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/networking.go#L34)
- [internal/provider/azure/networking.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/networking.go#L42)
- [internal/provider/azure/networking.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/networking.go#L132)
- [internal/provider/azure/networking.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/networking.go#L141)

### 2. Bootstrap creates those network resources inline

When bootstrapping a controller without a user-supplied VNet, the VM deployment includes
creation of the shared networking resources by calling `networkTemplateResources(...)`.

Relevant code:

- [internal/provider/azure/environ.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ.go#L727)

For non-controller models, the same shared network resources are created via the common
deployment path:

- [internal/provider/azure/environ.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ.go#L354)

### 3. Public IP creation explicitly forces IPv4

When the provider creates a VM public IP resource, it sets:

- `PublicIPAddressVersion: IPVersionIPv4`

Relevant code:

- [internal/provider/azure/environ.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ.go#L836)
- [internal/provider/azure/environ.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ.go#L852)

### 4. NIC creation never requests private IPv6

The VM creation path adds one NIC IP configuration per subnet, but that configuration only sets:

- `PrivateIPAllocationMethod: Dynamic`
- `Subnet: ...`

It does not set `PrivateIPAddressVersion: IPv6`, and it does not create a second IPv6 IP
configuration on the NIC.

Relevant code:

- [internal/provider/azure/environ.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ.go#L861)

### 5. Subnet discovery discards IPv6

When reading subnet information back from Azure, the provider explicitly picks an IPv4 CIDR and
ignores IPv6 prefixes:

- `// We only care about IPv4 addresses.`

Relevant code:

- [internal/provider/azure/environ_network.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ_network.go#L95)
- [internal/provider/azure/environ_network.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ_network.go#L102)

This behavior is covered by a test that supplies both an IPv6 and IPv4 prefix and expects the
IPv4 CIDR to be returned:

- [internal/provider/azure/environ_network_test.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ_network_test.go#L63)

The VM deployment tests also assert that created public IPs are IPv4:

- [internal/provider/azure/environ_test.go](/home/joseph/go/src/github.com/juju/juju/internal/provider/azure/environ_test.go#L1399)

## Can We Change Only Public IP Allocation?

No, not for the VM/NIC model Juju currently uses.

Azure requires:

- A public IPv6 address to attach to an IPv6 NIC IP configuration
- That IPv6 NIC IP configuration to live on a subnet with IPv6 address space

So a public-only change is insufficient. Juju would need cloud-local IPv6 as well.

## Minimum Changes Needed For Dual-Stack Controllers

If Juju continues to manage the VNet:

- Add IPv6 VNet address space and IPv6 subnet prefixes in `networking.go`
- Add an IPv6 NIC IP configuration in `environ.go`
- Add an IPv6 public IP resource and attach it to that IPv6 NIC IP configuration
- Add IPv6-aware NSG rules for controller/API access
- Stop filtering IPv6 out of subnet discovery

If the user supplies an existing dual-stack VNet:

- Juju could skip the VNet/subnet creation work
- But it would still need to add an IPv6 NIC IP configuration and attach an IPv6 public IP

## External References

Azure documentation checked for these conclusions:

- NIC/IP configuration behavior: https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/virtual-network-network-interface-addresses
- Dual-stack update flow for a VM: https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/add-dual-stack-ipv6-vm-portal
- IPv6 VNet overview: https://learn.microsoft.com/en-us/azure/virtual-network/ip-services/ipv6-overview
