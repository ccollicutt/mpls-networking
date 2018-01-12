# Microtik

This workshop is based on the Microtik documentation for a [Simple TE MPLS network](https://wiki.mikrotik.com/wiki/Manual:Simple_TE). Also more information can be found [here](https://wiki.mikrotik.com/wiki/Manual:TE_Tunnels).

## NOTE - Issue with CHR and MPLS

MPLS is not working yet with CHR it seems. At this point the lab attempts to configure MPLS, then drops back to plain Layer 3 with OSPF.

## Overview

Several routers are deployed into an OpenStack cloud. Neutron networks are used as links. The routers are configured to be able to support MPLS tunnels.

### OpenStack

These virtual routers are deploying into an OpenStack cloud. The cloud has a global MTU of 1500 and it can't be increased. The links/OpenStack networks are VXLAN based but the OS of the virtual machine would not know that. The cloud is setup in such a way that the VXLAN networks also have an MTU of 1500. The underlying cloud network can support higher MTUs, but based on the configuration of OpenStack in this cloud the tenant virtual machines can only have a max of 1500.

### MicroTik Version

Cloud Hosted Router 6.39.3 and 6.41.

### Network Diagram

![mpls netowrk diagram](/img/${MPLS_NAME}-microtik.jpg)

## Setup Networks

First, setup a few variables. This is so that it is easier to have multiple deployments.

```
export MPLS_NAME="mpls2"
# NOTE: Just first 3 octets
export MPLS_MGMT_NET="192.168.1"
export MPLS_MT_IMAGE="chr-6.41"
```

Create networks.

```
os network create ${MPLS_NAME}-mgmt --disable-port-security
os network create ${MPLS_NAME}-link1 --disable-port-security
os network create ${MPLS_NAME}-link2 --disable-port-security
os network create ${MPLS_NAME}-link3 --disable-port-security
os network create ${MPLS_NAME}-link4 --disable-port-security
os network create ${MPLS_NAME}-lan1 --disable-port-security
os network create ${MPLS_NAME}-lan2 --disable-port-security
```

Create subnets.

*NOTE: dhcp disabled on link networks.*

*NOTE: Gateway on ${MPLS_NAME}-mgmt-subnet is .250. Gateway has to be specified on links but won't actually be used. We are using .1 as the R1 IP on all networks.*

```
os subnet create --gateway=${MPLS_MGMT_NET}.250 --subnet-range=${MPLS_MGMT_NET}.0/24 --network ${MPLS_NAME}-mgmt ${MPLS_NAME}-mgmt-subnet
os subnet create --no-dhcp --gateway 192.168.10.250 --subnet-range=192.168.10.0/24 --network ${MPLS_NAME}-lan1 ${MPLS_NAME}-lan1-subnet
os subnet create --no-dhcp --gateway 192.168.20.250 --subnet-range=192.168.20.0/24 --network ${MPLS_NAME}-lan2 ${MPLS_NAME}-lan2-subnet
os subnet create --no-dhcp --gateway 172.16.1.250 --subnet-range=172.16.1.0/24 --network ${MPLS_NAME}-link1 ${MPLS_NAME}-link1-subnet
os subnet create --no-dhcp --gateway 172.16.2.250 --subnet-range=172.16.2.0/24 --network ${MPLS_NAME}-link2 ${MPLS_NAME}-link2-subnet
os subnet create --no-dhcp --gateway 172.16.3.250 --subnet-range=172.16.3.0/24 --network ${MPLS_NAME}-link3 ${MPLS_NAME}-link3-subnet
os subnet create --no-dhcp --gateway 172.16.4.250 --subnet-range=172.16.4.0/24 --network ${MPLS_NAME}-link4 ${MPLS_NAME}-link4-subnet
```

Create ports.

```
# R1
os port create --network ${MPLS_NAME}-mgmt --fixed-ip subnet=${MPLS_NAME}-mgmt-subnet,ip-address=${MPLS_MGMT_NET}.11 ${MPLS_NAME}-r1-mgmt
os port create --network ${MPLS_NAME}-link1 --fixed-ip subnet=${MPLS_NAME}-link1-subnet,ip-address=172.16.1.1 ${MPLS_NAME}-r1-link1
os port create --network ${MPLS_NAME}-link4 --fixed-ip subnet=${MPLS_NAME}-link4-subnet,ip-address=172.16.4.1 ${MPLS_NAME}-r1-link4
os port create --network ${MPLS_NAME}-lan1 --fixed-ip subnet=${MPLS_NAME}-lan1-subnet,ip-address=192.168.10.1 ${MPLS_NAME}-r1-lan1

# R2
os port create --network ${MPLS_NAME}-mgmt --fixed-ip subnet=${MPLS_NAME}-mgmt-subnet,ip-address=${MPLS_MGMT_NET}.12 ${MPLS_NAME}-r2-mgmt
os port create --network ${MPLS_NAME}-link1 --fixed-ip subnet=${MPLS_NAME}-link1-subnet,ip-address=172.16.1.2 ${MPLS_NAME}-r2-link1
os port create --network ${MPLS_NAME}-link2 --fixed-ip subnet=${MPLS_NAME}-link2-subnet,ip-address=172.16.2.2 ${MPLS_NAME}-r2-link2

# R3
os port create --network ${MPLS_NAME}-mgmt --fixed-ip subnet=${MPLS_NAME}-mgmt-subnet,ip-address=${MPLS_MGMT_NET}.13 ${MPLS_NAME}-r3-mgmt
os port create --network ${MPLS_NAME}-link2 --fixed-ip subnet=${MPLS_NAME}-link2-subnet,ip-address=172.16.2.3 ${MPLS_NAME}-r3-link2
os port create --network ${MPLS_NAME}-link3 --fixed-ip subnet=${MPLS_NAME}-link3-subnet,ip-address=172.16.3.3 ${MPLS_NAME}-r3-link3
os port create --network ${MPLS_NAME}-lan2 --fixed-ip subnet=${MPLS_NAME}-lan2-subnet,ip-address=192.168.20.3 ${MPLS_NAME}-r3-lan2

# R4
os port create --network ${MPLS_NAME}-mgmt --fixed-ip subnet=${MPLS_NAME}-mgmt-subnet,ip-address=${MPLS_MGMT_NET}.14 ${MPLS_NAME}-r4-mgmt
os port create --network ${MPLS_NAME}-link3 --fixed-ip subnet=${MPLS_NAME}-link3-subnet,ip-address=172.16.3.4 ${MPLS_NAME}-r4-link3
os port create --network ${MPLS_NAME}-link4 --fixed-ip subnet=${MPLS_NAME}-link4-subnet,ip-address=172.16.4.4 ${MPLS_NAME}-r4-link4

# Utility/Ansible node
# NOTE: .249
os port create --network ${MPLS_NAME}-mgmt --fixed-ip subnet=${MPLS_NAME}-mgmt-subnet,ip-address=${MPLS_MGMT_NET}.249 ${MPLS_NAME}-util-mgmt
os port create --network ${MPLS_NAME}-lan1 --fixed-ip subnet=${MPLS_NAME}-lan1-subnet,ip-address=192.168.10.249 ${MPLS_NAME}-client-lan1
os port create --network ${MPLS_NAME}-lan2 --fixed-ip subnet=${MPLS_NAME}-lan2-subnet,ip-address=192.168.20.249 ${MPLS_NAME}-client-lan2

```

## Create Routers

Boot virtual machine images: routers, util, and client nodes.

*NOTE: Microtik CHR image!*

```
openstack server create \
	--image ${MPLS_MT_IMAGE} \
	--flavor m1.small \
	--nic port-id=${MPLS_NAME}-r1-mgmt \
  --nic port-id=${MPLS_NAME}-r1-lan1 \
	--nic port-id=${MPLS_NAME}-r1-link1 \
  --nic port-id=${MPLS_NAME}-r1-link4 \
	--key-name default \
	${MPLS_NAME}-r1

openstack server create \
	--image ${MPLS_MT_IMAGE} \
	--flavor m1.small \
	--nic port-id=${MPLS_NAME}-r2-mgmt \
	--nic port-id=${MPLS_NAME}-r2-link1 \
  --nic port-id=${MPLS_NAME}-r2-link2 \
	--key-name default \
	${MPLS_NAME}-r2

openstack server create \
	--image ${MPLS_MT_IMAGE} \
	--flavor m1.small \
	--nic port-id=${MPLS_NAME}-r3-mgmt \
	--nic port-id=${MPLS_NAME}-r3-lan2 \
  --nic port-id=${MPLS_NAME}-r3-link2 \
  --nic port-id=${MPLS_NAME}-r3-link3 \
	--key-name default \
	${MPLS_NAME}-r3

openstack server create \
	--image ${MPLS_MT_IMAGE} \
	--flavor m1.small \
	--nic port-id=${MPLS_NAME}-r4-mgmt \
	--nic port-id=${MPLS_NAME}-r4-link3 \
  --nic port-id=${MPLS_NAME}-r4-link4 \
	--key-name default \
	${MPLS_NAME}-r4

openstack server create \
	--image xenial \
	--flavor m1.medium \
	--nic port-id=${MPLS_NAME}-util-mgmt \
	--key-name default \
	${MPLS_NAME}-util

openstack server create \
	--image xenial \
	--flavor m1.small \
	--nic port-id=${MPLS_NAME}-client-lan1 \
	--key-name default \
	${MPLS_NAME}-client-lan1

openstack server create \
	--image xenial \
	--flavor m1.small \
	--nic port-id=${MPLS_NAME}-client-lan2 \
	--key-name default \
	${MPLS_NAME}-client-lan2

```

## Configure Utility Node

TBD

## Configure MicroTik Routers

R1:

```
/interface bridge add name=Loopback
/ip address add address=192.168.10.1/24 interface=ether2
/ip address add address=172.16.1.1/24 interface=ether3
/ip address add address=172.16.4.1/24 interface=ether4
/ip address add address=10.255.0.1/32 interface=Loopback
```

R2:

```
/interface bridge add name=Loopback
/ip address add address=172.16.1.2/24 interface=ether2
/ip address add address=172.16.2.2/24 interface=ether3
/ip address add address=10.255.0.2/32 interface=Loopback
```

R3:

```
/interface bridge add name=Loopback
/ip address add address=192.168.20.3/24 interface=ether2
/ip address add address=172.16.2.3/24 interface=ether3
/ip address add address=172.16.3.3/24 interface=ether4
/ip address add address=10.255.0.3/32 interface=Loopback
```

R4:

```
/interface bridge add name=Loopback
/ip address add address=172.16.3.4/24 interface=ether2
/ip address add address=172.16.4.4/24 interface=ether3
/ip address add address=10.255.0.4/32 interface=Loopback
```

## Loopback address reachability and CSPF setup

R1:

*NOTE: The netmask for the backbone is /21.*

```
/routing ospf instance set default router-id=10.255.0.1 ${MPLS_NAME}-te-area=backbone ${MPLS_NAME}-te-router-id=Loopback
/routing ospf network add network=172.16.0.0/21 area=backbone
/routing ospf network add network=10.255.0.1/32 area=backbone
```

R2:

```
/routing ospf instance set default router-id=10.255.0.2 ${MPLS_NAME}-te-area=backbone ${MPLS_NAME}-te-router-id=Loopback
/routing ospf network add network=172.16.0.0/21 area=backbone
/routing ospf network add network=10.255.0.2/32 area=backbone
```

R3:

```
/routing ospf instance set default router-id=10.255.0.3 ${MPLS_NAME}-te-area=backbone ${MPLS_NAME}-te-router-id=Loopback
/routing ospf network add network=172.16.0.0/21 area=backbone
/routing ospf network add network=10.255.0.3/32 area=backbone
```

R4:

```
/routing ospf instance set default router-id=10.255.0.4 ${MPLS_NAME}-te-area=backbone ${MPLS_NAME}-te-router-id=Loopback
/routing ospf network add network=172.16.0.0/21 area=backbone
/routing ospf network add network=10.255.0.4/32 area=backbone
```

## Set Resource Allocation

R1:

```
/mpls traffic-eng interface add interface=ether3 bandwidth=10Mbps
/mpls traffic-eng interface add interface=ether4 bandwidth=10Mbps
```

```
[admin@${MPLS_NAME}-r1.novalocal] > mpls traffic-eng interface pr
Flags: X - disabled, I - invalid
 #   INTERFACE                                                                                                                                              BANDWIDTH  TE-METRIC REMAINING-BW
 0   ether2               10Mbps          1     10.0Mbps
 1   ether3               10Mbps          1     10.0Mbps
 2   ether4               10Mbps          1     10.0Mbps
[admin@${MPLS_NAME}-r1.novalocal] >
```

R2:

```
/mpls traffic-eng interface add interface=ether2 bandwidth=10Mbps
/mpls traffic-eng interface add interface=ether3 bandwidth=10Mbps
```

R3:

```
/mpls traffic-eng interface add interface=ether3 bandwidth=10Mbps
/mpls traffic-eng interface add interface=ether4 bandwidth=10Mbps
```

R4:

```
/mpls traffic-eng interface add interface=ether2 bandwidth=10Mbps
/mpls traffic-eng interface add interface=ether3 bandwidth=10Mbps
```

## Tunnel Setup

R1:


```
# primary
/mpls traffic-eng tunnel-path add name=tun-first-link use-cspf=no \
   hops=172.16.2.2:strict,172.16.3.2:strict,172.16.3.3:strict

# secondary
/mpls traffic-eng tunnel-path add name=dyn use-cspf=yes

/interface traffic-eng \
  add bandwidth=5Mbps \
  name=TE-to-R3 \
  disabled=no \
  to-address=10.255.0.3 \
  primary-path=tun-first-link \
  secondary-paths=dyn \
  record-route=yes \
  from-address=10.255.0.1


```

Show tunnel paths.

```
[admin@${MPLS_NAME}-r1.novalocal] > mpls traffic-eng tunnel-path print
Flags: X - disabled
 #   NAME                           USE-CSPF HOPS                  
 0   dyn                            yes     
 1   tun-first-link                 no       172.15.2.2:strict     
                                             172.16.3.2:strict     
                                             172.16.3.3:strict
```

R3:

```
/mpls traffic-eng tunnel-path add \
  name=tun-second-link \
  use-cspf=no \
  hops=172.16.5.4:strict,172.16.6.4:strict,172.16.6.1:strict

# secondary
/mpls traffic-eng tunnel-path add \
  name=dyn \
  use-cspf=yes

/interface traffic-eng add \
  bandwidth=5Mbps \
  name=TE-to-R1 \
  disabled=no \
  to-address=10.255.0.1 \
  primary-path=tun-second-link \
  secondary-paths=dyn \
  record-route=yes \
  from-address=10.255.0.3
```

Monitor the tunnel.

```
[admin@${MPLS_NAME}-r3.novalocal] > interface traffic-eng monitor 0
             tunnel-id: 1
    primary-path-state: established
          primary-path: tun-second-link
  secondary-path-state: not-necessary
           active-path: tun-second-link
          active-lspid: 1
          active-label: 16
        explicit-route: S:172.16.5.4/32,S:172.16.6.4/32,S:172.16.6.1/32
        recorded-route: 172.16.6.4[16],172.16.6.1[0]
    reserved-bandwidth: 5.0Mbps
```

## Route Traffic Over Tunnel

R1:

```
/ip address add address=10.99.99.1/30 interface=TE-to-R3
/ip route add dst-address=192.168.20.0/24 gateway=10.99.99.2
```

R3:

```
/ip address add address=10.99.99.2/30 interface=TE-to-R1
/ip route add dst-address=192.168.10.0/24 gateway=10.99.99.1
```

From R1 traceroute to the end of the tunnel.

```
/tool traceroute 10.99.99.2
```

## (Optional) LDP

RSVP-TE is taking care of this AFAIK.

*NOTE: Not sure this is necessary, was not part of the "Simple TE" instructions.*

R1:

```
/mpls ldp set enabled=yes lsr-id=10.255.0.1 transport-address=10.255.0.1
/mpls ldp interface add interface=ether3
/mpls ldp interface add interface=ether4
```

R2:

```
/mpls ldp set enabled=yes lsr-id=10.255.0.2 transport-address=10.255.0.2
/mpls ldp interface add interface=ether2
/mpls ldp interface add interface=ether3
```

R3:

```
/mpls ldp set enabled=yes lsr-id=10.255.0.3 transport-address=10.255.0.3
/mpls ldp interface add interface=ether3
/mpls ldp interface add interface=ether4
```

R4:

```
/mpls ldp set enabled=yes lsr-id=10.255.0.4 transport-address=10.255.0.4
/mpls ldp interface add interface=ether2
/mpls ldp interface add interface=ether3
```

## MTU

```
[admin@${MPLS_NAME}-r4.novalocal] > mpls interface print
Flags: X - disabled, * - default
 #    INTERFACE                                                                                                                                                                      ${MPLS_NAME}-MTU
 0  * all     
```

Set to 1450?

```
[admin@${MPLS_NAME}-r3.novalocal] > mpls interface set ${MPLS_NAME}-mtu=1450 0
[admin@${MPLS_NAME}-r3.novalocal] > mpls interface print
Flags: X - disabled, * - default
 #    INTERFACE                                                                                                                                                                      ${MPLS_NAME}-MTU
 0  * all        
 ```

## Drop Down to Plain OSPF Routing

If MPLS is not working, we can simply drop down to just routing with plain Layer 3.

* Remove the routes to the LAN1 and LAN2 networks from R1 and R3
* Remove the 10.99.99.0/30 IPs from the TE-* interfaces on R1 and R3
* Remove the traffic engineering

R1:

```
[admin@mpls2-r1.novalocal] > interface traffic-eng remove TE-to-R3
```

R3:

```
[admin@mpls2-r3.novalocal] > interface traffic-eng remove TE-to-R1
```

* Add the LAN1 and LAN2 networks to OSPF

R1:

```
[admin@mpls2-r1.novalocal] > /routing ospf network add network=192.168.10.0/24 area=backbone
```

R3:

```
[admin@mpls2-r1.novalocal] > /routing ospf network add network=192.168.20.0/24 area=backbone
```

* Add LAN{1,2}.250 as IPs on R1 and R3, as that is the gateway that was set when the subnets were created.

R1:

```
[admin@mpls2-r1.novalocal] > ip address add address=192.168.10.250/24 interface=ether2
```

R3:

```
[admin@mpls2-r3.novalocal] > ip address add address=192.168.20.250/24 interface=ether2   
```

From R3 we should be able to ping `192.168.10.249` which is client-lan2.

```
[admin@mpls2-r3.novalocal] > ping 192.168.10.250
  SEQ HOST                                     SIZE TTL TIME  STATUS                                                                                                                         
    0 192.168.10.250                             56  63 5ms  
    1 192.168.10.250                             56  63 1ms  
    2 192.168.10.250                             56  63 1ms  
    sent=3 received=3 packet-loss=0% min-rtt=1ms avg-rtt=2ms max-rtt=5ms
```

And from R1 to `192.168.20.249`:

```
[admin@mpls2-r1.novalocal] > ping 192.168.20.249
  SEQ HOST                                     SIZE TTL TIME  STATUS                                                                                                                         
    0 192.168.20.249                             56  62 1ms  
    1 192.168.20.249                             56  62 1ms  
    2 192.168.20.249                             56  62 1ms  
    sent=3 received=3 packet-loss=0% min-rtt=1ms avg-rtt=1ms max-rtt=1ms
```

And with that at least Layer 3 routing is working fine.

## Teardown

Delete servers.

```
export SERVERS="${MPLS_NAME}-r1             
${MPLS_NAME}-client-lan2    
${MPLS_NAME}-client-lan1    
${MPLS_NAME}-util           
${MPLS_NAME}-r4             
${MPLS_NAME}-r3             
${MPLS_NAME}-r2"

for s in ${SERVERS}; do
  os server delete $s
done
```

Delete ports.

```
export PORTS="${MPLS_NAME}-client-lan1
${MPLS_NAME}-client-lan2
${MPLS_NAME}-r1-lan1
${MPLS_NAME}-r1-link1
${MPLS_NAME}-r1-link4
${MPLS_NAME}-r1-mgmt
${MPLS_NAME}-r2-link1
${MPLS_NAME}-r2-link2
${MPLS_NAME}-r2-mgmt
${MPLS_NAME}-r3-lan2
${MPLS_NAME}-r3-link2
${MPLS_NAME}-r3-link3
${MPLS_NAME}-r3-mgmt
${MPLS_NAME}-r4-link3
${MPLS_NAME}-r4-link4
${MPLS_NAME}-r4-mgmt
${MPLS_NAME}-util-mgmt"

for p in $PORTS; do
 os port delete $p
done
```

Subnets:

```
export SUBNETS="${MPLS_NAME}-mgmt-subnet
${MPLS_NAME}-link1-subnet
${MPLS_NAME}-link2-subnet
${MPLS_NAME}-link3-subnet
${MPLS_NAME}-link4-subnet
${MPLS_NAME}-lan1-subnet
${MPLS_NAME}-lan2-subnet"

for s in $SUBNETS; do
  os subnet delete $s
done
```

Networks:

```
export NETS="${MPLS_NAME}-mgmt
${MPLS_NAME}-link1
${MPLS_NAME}-link2
${MPLS_NAME}-link3
${MPLS_NAME}-link4
${MPLS_NAME}-lan1
${MPLS_NAME}-lan2"

for n in $NETS; do
  os network delete $n
done
```

## Issues

* [CHR and MPLS not working on KVM?](https://forum.mikrotik.com/viewtopic.php?t=122446) and [here](https://forum.mikrotik.com/viewtopic.php?t=128396) (though points to the same forum post). That said it seems like a performance thing, not that it doesn't work at all.

## Trouble Shooting

Sniffer:

```
tool sniffer start
tool sniffer connection print interval=0.2
tool sniffer stop
```

## links

* [MPLS slides](https://mum.mikrotik.com//presentations/US16/presentation_3327_1462279781.pdf)
* [MPLS for ISPs - PPoE over VPLS](https://www.youtube.com/watch?v=Q8AF-Srulmk&feature=youtu.be)
* [MicroTik MPLS RSVP TE](https://wiki.mikrotik.com/wiki/Manual:TE_Tunnels)
