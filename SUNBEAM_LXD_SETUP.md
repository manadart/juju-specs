# Sunbeam LXD VM Notes

These are the local setup notes for the `sunbeam` LXD VM used to run a
single-node Sunbeam OpenStack for Juju/OpenStack testing. Secrets are not
recorded here; use the existing RC files on the machine.

## Current VM Shape

The VM is named `sunbeam` and is an LXD virtual machine based on Ubuntu
24.04 amd64.

Current resource shape:

- CPU: 4 cores
- Memory: 16 GiB
- Root disk: 120 GiB on LXD storage pool `sunbeam-pool`
- Extra block disk: `sunbeam-ceph`, 120 GiB, attached as `ceph-disk`
- LXD storage pool: `sunbeam-pool`, ZFS, 300 GiB backing file
- Primary host-facing network: `lxdbr0`, `10.155.5.0/24`
- Extra LXD network attached as `eth1`: `lxdbr1`, `10.155.6.0/24`

Useful inspection commands:

```bash
lxc list sunbeam
lxc config show sunbeam --expanded
lxc storage show sunbeam-pool
lxc storage volume show sunbeam-pool custom/sunbeam-ceph
```

Representative creation commands for a fresh VM:

```bash
lxc storage create sunbeam-pool zfs size=300GiB
lxc init ubuntu:24.04 sunbeam --vm --storage sunbeam-pool
lxc config set sunbeam limits.cpu 4
lxc config set sunbeam limits.memory 16GiB
lxc config device override sunbeam root size=120GiB
lxc storage volume create sunbeam-pool sunbeam-ceph --type=block size=120GiB
lxc config device add sunbeam ceph-disk disk pool=sunbeam-pool source=sunbeam-ceph
lxc start sunbeam
```

If a second NIC is needed for local testing:

```bash
lxc network create lxdbr1 ipv4.address=10.155.6.1/24 ipv4.nat=true ipv6.address=none
lxc config device add sunbeam eth1 nic network=lxdbr1
```

## Sunbeam Install

Inside the VM, Sunbeam is installed from the `openstack` snap:

```bash
lxc exec sunbeam -- snap list openstack
```

The current snap is `openstack` version `2024.1`, tracking
`2024.1/stable`, and it is held.

The VM has a generated demo credential at:

```text
/home/ubuntu/demo-openrc
```

Source that file inside the VM when using the OpenStack CLI as the demo
project user:

```bash
lxc exec sunbeam -- sudo -iu ubuntu -- bash -lc 'source /home/ubuntu/demo-openrc && openstack server list'
```

## Endpoints

The host-reachable Keystone endpoint used for the Juju cloud definition is:

```text
http://10.155.5.220:31984/openstack-keystone/v3
```

The Keystone endpoint inside the Sunbeam service catalog is:

```text
http://172.16.1.203/openstack-keystone/v3
```

Sunbeam's service catalog returns public service URLs on `172.16.1.0/24`,
for example:

```text
http://172.16.1.203:80/openstack-nova/v2.1
http://172.16.1.203:80/openstack-glance
http://172.16.1.203:80/openstack-neutron
```

The host therefore needs a route for that subnet through the LXD VM.

## Host Routes

The first bootstrap failure happened because Juju authenticated through the
NodePort Keystone URL, then followed catalog URLs on `172.16.1.203`, which
were not reachable from the host.

Add the catalog subnet route to the VM NIC:

```bash
lxc config device set sunbeam eth0 ipv4.routes 172.16.1.0/24
```

After bootstrap reached instance creation, Juju attempted to SSH to a tenant
address such as `192.168.0.64`. That is from Sunbeam's demo tenant network,
`192.168.0.0/24`, and it overlaps the host LAN. Do not route that subnet from
the host.

Use floating IPs instead. Sunbeam's external network is on `172.16.2.0/24`,
so the host also needs that subnet routed via the VM:

```bash
lxc config device set sunbeam eth0 ipv4.routes 172.16.1.0/24,172.16.2.0/24
```

For an LXD VM, those host routes are installed as link routes on `lxdbr0`.
The VM must answer ARP for addresses behind it, so proxy ARP is enabled on
the VM's host-facing NIC:

```bash
lxc exec sunbeam -- sysctl -w net.ipv4.conf.enp5s0.proxy_arp=1
lxc exec sunbeam -- sysctl -w net.ipv4.conf.enp5s0.forwarding=1
```

The persistent config is:

```text
/etc/sysctl.d/99-sunbeam-lxd-routing.conf
```

with:

```text
net.ipv4.ip_forward = 1
net.ipv4.conf.enp5s0.forwarding = 1
net.ipv4.conf.enp5s0.proxy_arp = 1
```

Sanity checks:

```bash
ip route get 172.16.1.203
ip route get 172.16.2.203
curl -I http://172.16.1.203:80/openstack-nova/
```

## OpenStack Networks

The useful Sunbeam networks are:

- `demo-network`: tenant network, subnet `192.168.0.0/24`
- `external-network`: external/floating-IP network, router gateway
  `172.16.2.203`

Inspection commands:

```bash
lxc exec sunbeam -- sudo -iu ubuntu -- bash -lc 'source /home/ubuntu/demo-openrc && openstack network list'
lxc exec sunbeam -- sudo -iu ubuntu -- bash -lc 'source /home/ubuntu/demo-openrc && openstack subnet list'
lxc exec sunbeam -- sudo -iu ubuntu -- bash -lc 'source /home/ubuntu/demo-openrc && openstack router show demo-router'
```

For Juju bootstrap, set the Neutron networks explicitly and request a floating
IP for the bootstrap machine:

```bash
juju bootstrap sunbeam/RegionOne sunbeam \
  --credential admin \
  --metadata-source /home/joseph/shed/sunbeam-image-metadata \
  --bootstrap-base ubuntu@24.04 \
  --bootstrap-constraints "arch=amd64 allocate-public-ip=true" \
  --constraints "arch=amd64 allocate-public-ip=true" \
  --config network=demo-network \
  --config external-network=external-network \
  --model-default network=demo-network \
  --model-default external-network=external-network \
  --debug
```

The important parts for the SSH issue are:

- `allocate-public-ip=true`
- `external-network=external-network`
- routing `172.16.2.0/24` from the host to the VM

## Juju Cloud And Trust Credential

The local Juju cloud is named `sunbeam`:

```bash
juju show-cloud sunbeam --client --format yaml
```

Expected cloud endpoint:

```text
http://10.155.5.220:31984/openstack-keystone/v3
```

The local Juju credential is a Keystone v3 userpass credential using a trust:

```bash
juju credentials sunbeam --client --format yaml
```

The trust-based shell RC file for direct auth testing is:

```text
/home/joseph/shed/sunbeam-trust.rc
```

It sets `OS_TRUST_ID` and intentionally unsets project scope variables. Do not
copy its password into this repository.

The trust created for testing delegates the demo project role to the admin
trustee. The trust ID currently in use is:

```text
b39ea145cd3b48328eabdccbee47034b
```

## Image Metadata

Juju did not find public simplestreams metadata for this private Sunbeam cloud,
so we generated local image metadata for the uploaded `ubuntu` image.

Current image:

```text
Name: ubuntu
ID: 5975f0ee-4445-467d-bd3f-139d394a5033
Base: ubuntu@24.04
Arch: amd64
```

Generation command:

```bash
mkdir -p /tmp/sunbeam-image-metadata
juju metadata generate-image \
  -d /tmp/sunbeam-image-metadata \
  -i 5975f0ee-4445-467d-bd3f-139d394a5033 \
  -r RegionOne \
  -u http://10.155.5.220:31984/openstack-keystone/v3 \
  --base ubuntu@24.04 \
  -a amd64 \
  --stream released
```

Persistent copy:

```bash
install -d -m 0755 /home/joseph/shed/sunbeam-image-metadata/images/streams/v1
cp -a /tmp/sunbeam-image-metadata/images /home/joseph/shed/sunbeam-image-metadata/
```

Validation command:

```bash
juju metadata validate-images \
  -p openstack \
  -d /home/joseph/shed/sunbeam-image-metadata \
  -r RegionOne \
  -u http://10.155.5.220:31984/openstack-keystone/v3 \
  --base ubuntu@24.04 \
  --stream released
```

## Failure Modes Seen

- `dial tcp 172.16.1.203:80: i/o timeout`: the host could not reach the
  Sunbeam catalog endpoint subnet. Add `172.16.1.0/24` to `eth0`
  `ipv4.routes` on the LXD VM.
- `model "controller" of type openstack does not support instances running on
  "amd64"`: Juju had no matching image metadata for this private cloud.
  Generate and pass local image metadata with `--metadata-source`.
- `ssh: connect to host 192.168.0.x port 22: No route to host`: Juju was
  trying to use the tenant IP. For this Sunbeam VM, the tenant network overlaps
  the host LAN, so use floating IPs via `allocate-public-ip=true`, set
  `external-network=external-network`, and route `172.16.2.0/24` through the
  VM.
- `ssh: connect to host 172.16.2.x port 22: No route to host`: a floating IP
  was allocated, but the VM was not answering ARP for the routed external
  subnet on `lxdbr0`. Enable proxy ARP on `enp5s0` in the Sunbeam VM.
