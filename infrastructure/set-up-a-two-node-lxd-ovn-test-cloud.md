---
myst:
  html_meta:
    description: "Build a reproducible two-node LXD cluster with OVN for Juju provider development and failure-domain testing."
---

(set-up-two-node-lxd-ovn-test-cloud)=
# Set up a two-node LXD OVN test cloud

This guide creates a nested, two-member LXD cluster for Juju provider
development. Each LXD member runs in a separate VM, and Juju instances use an
OVN logical network spanning both members.

This is deliberately a two-node test topology. It provides separate member
failure domains, but it does not provide production control-plane high
availability. LXD assigns `lxd-cloud-0` the database leader role and
`lxd-cloud-1` the database standby role. The standalone OVN central services
also run on `lxd-cloud-0`, so losing that member interrupts control-plane
operations until it recovers.

```{ibnote}
See also: [LXD | How to form a cluster](https://documentation.ubuntu.com/lxd/stable-5.21/howto/cluster_form/),
[LXD | How to set up OVN](https://documentation.ubuntu.com/lxd/stable-5.21/howto/network_ovn_setup/)
```

The commands below use the following addresses. Substitute non-conflicting
addresses when the host LXD bridge uses another subnet.

| Purpose | Value |
| --- | --- |
| Host LXD bridge | `lxdbr0`, `10.246.27.1/24` |
| `lxd-cloud-0` management and API | `10.246.27.20` |
| `lxd-cloud-1` management | `10.246.27.156` |
| OVN instance external pool | `10.246.27.230-10.246.27.239` |
| OVN router address pool | `10.246.27.240-10.246.27.249` |
| OVN instance subnet | `10.248.0.0/24` |
| Juju cloud endpoint | `https://10.246.27.20:8443` |

The management NIC and OVN uplink NIC are both attached to the host's
`lxdbr0`. The uplink is a raw bridged attachment on VLAN 1, so it shares the
working L2 and NAT path without registering a second DHCP or DNS identity for
the VM.

## Create the LXD member VMs

To confirm that the names and OVN address range are unused:

```text
lxc list lxd-cloud
lxc network list-leases lxdbr0
```

Reserve the OVN instance and router pools by excluding them from the host
bridge's DHCP range. The following range preserves addresses `2-229` and
`250-254`:

```text
lxc network set lxdbr0 \
  ipv4.dhcp.ranges=10.246.27.2-10.246.27.229,10.246.27.250-10.246.27.254
```

Create a profile with a 30 GiB root disk, a managed management NIC, and a raw
OVN uplink NIC:

```text
lxc profile create lxd-ovn-cloud-host
lxc profile device add lxd-ovn-cloud-host root disk \
  path=/ pool=default size=30GiB
lxc profile device add lxd-ovn-cloud-host eth0 nic \
  network=lxdbr0 name=eth0
lxc profile device add lxd-ovn-cloud-host eth1 nic \
  nictype=bridged parent=lxdbr0 name=eth1 vlan=1
```

Create the VMs. An already cached Ubuntu 24.04 VM image can be used instead of
the `ubuntu:24.04` alias.

```text
lxc init ubuntu:24.04 lxd-cloud-0 --vm \
  --profile lxd-ovn-cloud-host \
  -c limits.cpu=2 -c limits.memory=4GiB
lxc init ubuntu:24.04 lxd-cloud-1 --vm \
  --profile lxd-ovn-cloud-host \
  -c limits.cpu=2 -c limits.memory=4GiB
```

Pin the management addresses so that the cluster and Juju API endpoints remain
stable across reboots:

```text
lxc config device override lxd-cloud-0 eth0 ipv4.address=10.246.27.20
lxc config device override lxd-cloud-1 eth0 ipv4.address=10.246.27.156
lxc start lxd-cloud-0
lxc start lxd-cloud-1
lxc exec lxd-cloud-0 -- cloud-init status --wait
lxc exec lxd-cloud-1 -- cloud-init status --wait
```

On these VMs, the two NICs appear as `enp5s0` and `enp6s0`. Confirm the names
before using `enp6s0` as the OVN uplink:

```text
lxc exec lxd-cloud-0 -- ip -brief address
lxc exec lxd-cloud-1 -- ip -brief address
```

If the host's snap launcher cannot run `lxc`, substitute
`/snap/lxd/current/bin/lxc` for the host-side `lxc` commands.

## Install LXD and OVN

Install OVN central and chassis services on `lxd-cloud-0`, and only the chassis
service on `lxd-cloud-1`:

```text
lxc exec lxd-cloud-0 -- apt-get update
lxc exec lxd-cloud-1 -- apt-get update
lxc exec lxd-cloud-0 -- env DEBIAN_FRONTEND=noninteractive \
  apt-get install -y ovn-central ovn-host
lxc exec lxd-cloud-1 -- env DEBIAN_FRONTEND=noninteractive \
  apt-get install -y ovn-host
lxc exec lxd-cloud-0 -- systemctl enable ovn-central ovn-host
lxc exec lxd-cloud-1 -- systemctl enable ovn-host
```

Install the same LXD LTS release on both members:

```text
lxc exec lxd-cloud-0 -- snap install lxd --channel=5.21/stable
lxc exec lxd-cloud-1 -- snap install lxd --channel=5.21/stable
```

## Form the LXD cluster

Initialize `lxd-cloud-0` with local `dir` storage, expose its API, and enable
clustering:

```text
lxc exec lxd-cloud-0 -- /snap/bin/lxd init --minimal
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc config set \
  core.https_address 10.246.27.20:8443
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc cluster enable \
  lxd-cloud-0
```

Generate a single-use token and join `lxd-cloud-1`. The `tail` command selects
the token from the labelled output of `lxc cluster add`.

```text
JOIN_TOKEN="$(lxc exec lxd-cloud-0 -- \
  /snap/lxd/current/bin/lxc cluster add lxd-cloud-1 | tail -n 1)"

printf 'cluster:\n  enabled: true\n  server_address: 10.246.27.156:8443\n  cluster_token: %s\n  member_config:\n  - entity: storage-pool\n    name: default\n    key: source\n    value: ""\n' \
  "$JOIN_TOKEN" | lxc exec lxd-cloud-1 -- /snap/bin/lxd init --preseed

unset JOIN_TOKEN
```

To verify both members are online:

```text
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc cluster list
```

## Configure OVN

Expose the standalone northbound and southbound databases on
`lxd-cloud-0`:

```text
lxc exec lxd-cloud-0 -- ovn-nbctl set-connection \
  ptcp:6641:10.246.27.20
lxc exec lxd-cloud-0 -- ovn-sbctl set-connection \
  ptcp:6642:10.246.27.20
```

Register both Geneve chassis with the southbound database:

```text
lxc exec lxd-cloud-0 -- ovs-vsctl set open_vswitch . \
  external_ids:ovn-remote=tcp:10.246.27.20:6642 \
  external_ids:ovn-encap-type=geneve \
  external_ids:ovn-encap-ip=10.246.27.20
lxc exec lxd-cloud-1 -- ovs-vsctl set open_vswitch . \
  external_ids:ovn-remote=tcp:10.246.27.20:6642 \
  external_ids:ovn-encap-type=geneve \
  external_ids:ovn-encap-ip=10.246.27.156
```

Tell LXD how to reach the OVN northbound database:

```text
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc config set \
  network.ovn.northbound_connection tcp:10.246.27.20:6641
```

Define the member-specific physical interface and then finalize the clustered
uplink network:

```text
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc network create UPLINK \
  --type=physical parent=enp6s0 --target=lxd-cloud-0
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc network create UPLINK \
  --type=physical parent=enp6s0 --target=lxd-cloud-1
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc network create UPLINK \
  --type=physical \
  ipv4.gateway=10.246.27.1/24 \
  ipv4.ovn.ranges=10.246.27.240-10.246.27.249 \
  ipv4.routes=10.246.27.230/31,10.246.27.232/29 \
  dns.nameservers=1.1.1.1
```

The uplink routes form a ten-address pool for OVN instance ingress. The Juju
LXD provider asks LXD to allocate a network forward from this pool for each OVN
NIC. LXD forwards that address to the instance's DHCP address, making SSH and
controller API traffic reachable while preserving the OVN network's normal
outbound NAT.

Create the logical OVN network and make it the default for instances:

```text
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc network create juju-ovn \
  --type=ovn network=UPLINK \
  ipv4.address=10.248.0.1/24 ipv4.nat=true ipv6.address=none
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc profile device set \
  default eth0 network=juju-ovn
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc network delete lxdbr0
```

The last command removes only the nested bootstrap bridge. It does not remove
the host's `lxdbr0`.

To verify OVN and its chassis bindings:

```text
lxc exec lxd-cloud-0 -- ovn-nbctl show
lxc exec lxd-cloud-0 -- ovn-sbctl show
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc network show juju-ovn
```

The `juju-ovn` output should show `10.248.0.1/24` and a volatile external
address from `10.246.27.240-249`.

## Validate cross-member OVN traffic

Launch one disposable container on each member:

```text
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc launch \
  ubuntu:24.04 ovn-test-0 --target=lxd-cloud-0
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc launch \
  ubuntu:24.04 ovn-test-1 --target=lxd-cloud-1
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc list ovn-test \
  --format csv -c n4Ls
```

For the address allocation above, validate Geneve traffic in both directions,
outbound NAT, and DNS:

```text
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc exec ovn-test-0 -- \
  ping -c 3 10.248.0.3
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc exec ovn-test-1 -- \
  ping -c 3 10.248.0.2
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc exec ovn-test-0 -- \
  ping -c 3 1.1.1.1
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc exec ovn-test-1 -- \
  getent ahostsv4 ubuntu.com
```

Remove the disposable containers after the checks pass:

```text
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc delete \
  ovn-test-0 ovn-test-1 --force
```

## Add the cloud and credential to Juju

Create a private directory for a dedicated LXD client certificate:

```text
install -d -m 0700 ~/.local/share/juju/lxd-ovn-cluster
openssl req -x509 -newkey rsa:4096 -sha256 -days 3650 -nodes \
  -subj /CN=juju-lxd-ovn-cluster \
  -addext keyUsage=digitalSignature,keyEncipherment \
  -addext extendedKeyUsage=clientAuth \
  -keyout ~/.local/share/juju/lxd-ovn-cluster/client.key \
  -out ~/.local/share/juju/lxd-ovn-cluster/client.crt
chmod 0600 ~/.local/share/juju/lxd-ovn-cluster/client.key
```

Add the public certificate to the cluster trust store and save the cluster's
server certificate:

```text
lxc file push ~/.local/share/juju/lxd-ovn-cluster/client.crt \
  lxd-cloud-0/root/juju-lxd-ovn-cluster.crt
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc config trust add \
  /root/juju-lxd-ovn-cluster.crt --name juju-lxd-ovn-cluster
lxc file pull lxd-cloud-0/var/snap/lxd/common/lxd/cluster.crt \
  ~/.local/share/juju/lxd-ovn-cluster/server.crt
lxc file delete lxd-cloud-0/root/juju-lxd-ovn-cluster.crt
```

Save the following as
`~/.local/share/juju/lxd-ovn-cluster/cloud.yaml`:

```text
clouds:
  lxd-ovn-cluster:
    type: lxd
    auth-types:
      - certificate
    endpoint: https://10.246.27.20:8443
```

Save the following as
`~/.local/share/juju/lxd-ovn-cluster/credentials.yaml`:

```text
credentials:
  lxd-ovn-cluster:
    lxd-ovn-cluster:
      auth-type: certificate
      server-cert: ~/.local/share/juju/lxd-ovn-cluster/server.crt
      client-cert: ~/.local/share/juju/lxd-ovn-cluster/client.crt
      client-key: ~/.local/share/juju/lxd-ovn-cluster/client.key
```

Restrict the credential definition because it identifies private-key
material, even though Juju expands the paths when importing it:

```text
chmod 0600 ~/.local/share/juju/lxd-ovn-cluster/credentials.yaml
```

Register the cloud and credential with the local Juju client:

```text
juju add-cloud --client lxd-ovn-cluster \
  ~/.local/share/juju/lxd-ovn-cluster/cloud.yaml
juju add-credential --client lxd-ovn-cluster \
  -f ~/.local/share/juju/lxd-ovn-cluster/credentials.yaml
juju default-credential lxd-ovn-cluster lxd-ovn-cluster
```

To verify the definitions and certificate authentication:

```text
juju show-cloud --client lxd-ovn-cluster
juju credentials --client
curl --insecure \
  --cert ~/.local/share/juju/lxd-ovn-cluster/client.crt \
  --key ~/.local/share/juju/lxd-ovn-cluster/client.key \
  https://10.246.27.20:8443/1.0
```

The LXD response should report `"auth":"trusted"` and
`"server_clustered":true`. The `curl` command verifies client authentication;
Juju separately validates and stores the server certificate supplied in the
credential definition. For a cluster, this must be `cluster.crt`, because that
is the certificate presented by the HTTPS endpoint.

## Bootstrap Juju

Build the current tree and bootstrap without changing the active controller:

```text
make install
~/go/bin/juju bootstrap lxd-ovn-cluster lxd-ovn-proto --no-switch --debug
```

Confirm that the controller is healthy and that its address has been
forwarded through OVN:

```text
~/go/bin/juju status -m lxd-ovn-proto:controller
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc list \
  --format csv -c n4Ls
lxc exec lxd-cloud-0 -- /snap/lxd/current/bin/lxc network forward list \
  juju-ovn
```

The forward's default target is the controller's `10.248.0.0/24` DHCP address.
Its listen address comes from `10.246.27.230-239` and should answer on ports 22
and 17070 after the controller is ready.

## Remove the test cloud

To remove the Juju client definitions:

```text
juju remove-credential --client lxd-ovn-cluster lxd-ovn-cluster
juju remove-cloud --client lxd-ovn-cluster
```

To remove the two VMs and their host profile:

```text
lxc delete lxd-cloud-0 lxd-cloud-1 --force
lxc profile delete lxd-ovn-cloud-host
```

If no other test uses the reserved OVN range, restore the host bridge's default
DHCP allocation behavior:

```text
lxc network unset lxdbr0 ipv4.dhcp.ranges
```

Finally, remove `~/.local/share/juju/lxd-ovn-cluster` if its certificates and
YAML definitions are no longer required.
