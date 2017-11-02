# L3VPN setup 

This is sample steps for a two node L3VPN with two VPP instances with GoBGP and FIB agent running for each of the VPP instances.
VPN VRF name : "vrf1"
This is done in a single Ubuntu 16.04 VM 

                         (1.1.1.1)                         (2.2.2.2)
                         GoBGP1                            GoBGP2
                          |                                 |			 
                         GoRIB1                           GoRIB2
                          |                                 |			 
 ---- 11.1.1.0/24 (vrf1)-VPP1(10.1.2.1)--global--(10.1.2.2)VPP2-(vrf1)22.1.1.0/24)
	                 


## Start VPP instances

sudo vpp api-segment { prefix vpp1 }
sudo vpp api-segment { prefix vpp2 }

## Start GoRIB agents

- GoRIB1 on vpp1
sudo ./gorib --grpc-serv ":50055" --vpp vpp1

- GoRIB2 on vpp2
sudo ./gorib --grpc-serv ":50056" --vpp vpp2

## Start GoBGP deamons

./gobgpd --api-hosts 127.0.0.1:50051 -f bgp-1-loopback.cfg &
./gobgpd --api-hosts 127.0.0.1:50052 -f bgp-2-loopback.cfg &
./gobgpd --api-hosts 127.0.0.1:50025 -f bgp-con-loopback.cfg &


sudo ip link add name bgp12 type veth peer name bgp21
sudo ip link set dev bgp12 up
sudo ip link set dev bgp21 up
sudo ip addr add 10.1.2.1/24 dev bgp12
sudo ip addr add 10.1.2.2/24 dev bgp21


## VPP1 to host connection

sudo ip link add name vpp1out type veth peer name vpp1host
sudo ip link set dev vpp1out up
sudo ip link set dev vpp1host up
sudo ip addr add 11.1.1.2/24 dev vpp1host
sudo vppctl -p vpp1 create host-interface name vpp1out
sudo vppctl -p vpp1 set int state host-vpp1out up


## VPP2 to host connection

sudo ip link add name vpp2out type veth peer name vpp2host
sudo ip link set dev vpp2out up
sudo ip link set dev vpp2host up
sudo ip addr add 22.1.1.2/24 dev vpp2host
sudo vppctl -p vpp2 create host-interface name vpp2out
sudo vppctl -p vpp2 set int state host-vpp2out up


## VPP1 to VPP2 connection (global VRF)

sudo ip link add name vpp1vpp2 type veth peer name vpp2vpp1
sudo ip link set dev vpp1vpp2 up
sudo ip link set dev vpp2vpp1 up

sudo vppctl -p vpp1 create host-interface name vpp1vpp2
sudo vppctl -p vpp1 set int state host-vpp1vpp2 up
sudo vppctl -p vpp2 create host-interface name vpp2vpp1
sudo vppctl -p vpp2 set int state host-vpp2vpp1 up

## Loopbacks 

sudo ip link add name loop12 type veth peer name loop21
sudo ip link set dev loop12 up
sudo ip link set dev loop21 up
sudo ip addr add 1.1.1.1/32 dev loop12
sudo ip addr add 2.2.2.2/32 dev loop21

vppctl -p vpp1 loopback create-interface
vppctl -p vpp1 set int state loop0 up
vppctl -p vpp2 loopback create-interface
vppctl -p vpp2 set int state loop0 up


## RD/RT configuration for VRF vrf1 in BGP

./gobgp -p 50051 vrf add vrf1 id 5 rd 10.100:100 rt both 10.100:100 import 10.100:100 export 10.100:100
./gobgp -p 50052 vrf add vrf1 id 5 rd 10.100:100 rt both 10.100:100 import 10.100:100 export 10.100:100


## Host network address configuration FIB agent 1 (VPP1) at VRF1
./gorib_cli  --port 50055 interface set name vpp1vpp2 address 10.1.2.1/24 vrf global
./gorib_cli  --port 50056 interface set name vpp2vpp1 address 10.1.2.2/24 vrf global

## Host networks (loopbacks) with SR indexes

./gorib_cli  --port 50055 interface set name loop0 address 1.1.1.1/32 vrf global sid-index 1 srgb-base 2000 srgb-range 1000
./gorib_cli  --port 50056 interface set name loop0 address 2.2.2.2/32 vrf global  sid-index 2 srgb-base 2000 srgb-range 1000

## VRF networks

./gorib_cli  --port 50055 interface set name vpp1out address 11.1.1.1/24 vrf vrf1
./gorib_cli  --port 50056 interface set name vpp2out2 address 44.1.1.1/24 vrf vrf1 

## GoBGP SRTE policies


### SRTE policies to 2.2.2.2 color 200 from Controller

./gobgp -p 50025 tep add -a ipv4-srte distinguisher 100 color 200 endpoint 2.2.2.2 rt 1.1.1.1:0  tunnel-encap srte bsid 5855 preference 10 weight 4 label-stack 2002 2003 

### Pop the label at vpp2 for ping to work
vppctl -p vpp2 mpls local-label 2003 ip4-lookup-in-table 0 eos
vppctl -p vpp2 mpls local-label 2003 mpls-lookup-in-table 0

## VRF networks with a color

./gorib_cli  --port 50056 interface set name vpp2out address 22.1.1.1/24 vrf vrf1 color 200


## show commands

### show mpls fib
show mpls fib <table>  <label>
eg. show mpls fib 0 1168

### ping VRF in VPP
@vpp2: 
	ping 11.1.1.1 source host-vpp2out2
@vpp1: 
	ping 44.1.1.1 source host-vpp1out
	ping 22.1.1.1 source host-vpp1out

### GoBGP show
    GoBGP 1
	gobgp1 global rib -a vpnv4
	gobgp1 -p 50051 vrf vrf1 rib 

	./gobgp -p 50051 vrf vrf1 rib 
	./gobgp -p 50051 global rib -a vpnv4
	./gobgp -p 50051 neighbor 10.1.2.2
    GoBGP 2
	gobgp2 global rib -a vpnv4
	gobgp2 -p 50051 vrf vrf1 rib 

	./gobgp -p 50052 vrf vrf1 rib
	./gobgp -p 50052 global rib -a vpnv4
	./gobgp -p 50052 neighbor 10.1.2.1




## Tracing in VPP

sudo vppctl -p vpp1 trace add af-packet-input 10000
sudo vppctl -p vpp2 trace add af-packet-input 10000
sudo vppctl -p vpp3 trace add af-packet-input 10000
sudo vppctl -p vpp1 trace add dpdk-input 1000
sudo vppctl -p vpp2 trace add dpdk-input 1000
sudo vppctl -p vpp3 trace add dpdk-input 1000
sudo vppctl -p vpp1 clear trace
sudo vppctl -p vpp2 clear trace
sudo vppctl -p vpp3 clear trace

