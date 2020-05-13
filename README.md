# GoBGP, and GoRIB over VPP(fd.io)

## Overview

This package contains binaries for GoBGP, GoRIB(FIB agent), GoBGP-CLI, FIB agent-CLI.

Following features are added/verified end-to-end (GoBGP <-> FIB agent <-> VPP)
with traffic

    1. IPv4 routes

    2. IPv4 - MPLS
    	-Local routes configured to FIB agent/VPP are advertised by BGP as labeled unicast.
        -Received LU routes are programmed to VPP 

    3. IPv4 - Segment Routing with MPLS encapsulation (BGP Prefix SID extensions)

    4. IPv4 - SRTE (Segment Routing Traffic Engineering) (BGP SR TE)
    	-MPLS TE tunnels with label stack are programmed to VPP for SRTE policies.	
    	-Integration with SRTE APIs of VPP APIs are progressing well and is disabled now in these binaries, as the VPP SRTE APIs are not yet available in VPP (fd.io) repository.

    5. L3VPN
       -Local VRF networks configured are advertised by BGP as VPN-unicast routes with VPN labels.
       -Color extended community support in GoBGP

### GoBGP 
[GoBGP] (https://github.com/osrg/gobgp)  
GoBGP (Apache 2.0) is enhanced with Segment Routing (SR, SRTE), GRPC southbound programming infrastructure, Programming IP, MPLS, SR, SR-TE, VPN (L3VPN) to FIB agent via GRPC. 

### GoRIB
GoRIB contains, RIB, LFIB for basic route resolution functionality.
GoRIB contains a gRPC server. A gRPC client (eg. goBGP) can program the FIB agent via the gRPC APIs. 
When a gRPC client registers to FIB agent, it connects to the forwarder plugin (eg. Go VPP). 
GoRIB translates the gRPC API into VPP API after it does basic resoltion on certain routes.
GoRIB also has APIs to provision interfaces and interface addresses to VPP.
We recommend interface address configuration via FIB agent to support certain route resoltion that VPP FIB cannot perform.

Networks/Routes programmed via FIB agent is advertised to GoBGP for advertisements.
Networks/Routes in a VRF are advertised as VPN routes with VPN labels.
Networks/Routes in a global VRF are advertised as Labeled unicast routes with labels.

GoRIB reads all the interfaces from VPP in the beginning or whenever an IP address is assigned to an interface via FIB agent. 
IP address on an interface are programmed FIB agent CLI (executable).
FIB agent prgrams the interface addresses in VPP.

## Environment
Binaries are built in Ubuntu 16.04.3 LTS (Xenial) 64bit

## Binaries
	binaries:
		gobgp  -> GoBGP command line
		gobgpd -> GoBGP daemon
		agent_cli -> FIB agent CLI
		fib-agent -> FIB agent

### gobgp 
This binary is GoBGP Command Line Interface

Examples:
https://github.com/osrg/gobgp/blob/master/docs/sources/cli-command-syntax.md

Sample examples:

./gobgp global rib add -a ipv4 10.0.0.0/24 nexthop 20.20.20.20
./gobgp global rib add -a ipv4-mpls 10.0.0.0/24 100 nexthop 20.20.20.20

Additional CLIs

SRTE: 
./gobgp tep add -a ipv4-srte distinguisher 100 color 200 endpoint 1.1.1.1 rt 1.1.1.1:0  tunnel-encap srte bsid 5855 preference 10 weight 4 label-stack 100 200 300 400

SR(Prefix SID)

./gobgp  global rib add -a ipv4-mpls 1.1.1.1/32 2001  prefix-sid label-index 1 srgb 2000 1000

./gobgp global rib add -a ipv4-mpls 10.0.0.0/24 100 nexthop 20.20.20.20 color 100 

Show commands
./gobgp global rib -a ipv4
./gobgp global rib -a ipv4-mpls
./gobgp global rib -a ipv4-srte


### gobgpd: GoBGP daemon
Sample config is copied in configs directory.
Run the BGP daemon => ./gobgpd -f ../configs/bgp_agent-1.1.1.1.cfg

If gobgpd needs to talk to GoRIB or Fibagent to program VPP, you need to set following in the configuration.
If gobgpd needs to run standalone, don't use this configuration.

GRPC client ("fibagent") configuration can be specified in the configuration file.
[fibagent]
    [fibagent.config]
	enabled = true
	url = "127.0.0.1:50055"       <--- FIB agent target "IP:Port"

### GoRIB or FIB agent

GoRIB/FIB agent contains, RIB, LFIB for basic route resolution functionality.
It stores routes on each VRFs.
FIB agent contains a gRPC server. A gRPC client (eg. goBGP) can program the FIB agent via the gRPC APIs. 
When a gRPC client registers to FIB agent, it connects to the forwarder plugin (eg. Go VPP). 
FIB agent translates the gRPC API into VPP API after it does basic resoltion on certain routes.
FIB agent also has APIs to provision interfaces and interface addresses to VPP.
We recommend interface address configuration via FIB agent to support certain route resoltion that VPP FIB cannot perform.
FIB agent reads all the interfaces from VPP in the beginning or whenever an IP address is assigned to an interface via FIB agent. 
For configuring an IP address on an interface FIB agent CLI (executable) can be used.

Fib agent can be started with default options.
	sudo ./fibagent 

If there are no options speficied, FIB agent is started with a gRPC server listening to port 50055
FIB agent can be started with a specific gRPC IP/Port.

sudo ./fibagent --grpc-serv ":50056"  
   or
sudo ./fibagent --grpc-serv "127.0.0.1:50056"  


#### FIB agent connecting to  VPP instances
By default FIB agent connects to VPP service.
If there are multiple VPP instances, FIB agent can be connected to a specific VPP instance.
--vpp keyword can specified to refer to a VPP instance.
For example: Connect to VPP instance vpp1

sudo ./fibagent --grpc-serv ":50056"  --vpp vpp1 

This commands runs FIB agent with GRPC server port 50056 and connecting to VPP instance vpp1  

### agent_cli: CLI for FIB agent
agent_cli is a simple gRPC client and talks gRPC API to FIB agent. 
At present it supports only interface address provisioning on VPP interfaces. 
We are adding support for rest of the gRPC APIs via this CLI.

Interface address needs to programmed via FIB agent CLI.
This will help route resolution of connected routes (especially MPLS).

These routes will be given GoBGP for advertising. 
"global" VRF routes are advertised as labeled unicast routes by GoBGP to its neighbors.
VRF routes are advertised as VPN-unicast-MPLS routes by GoBGP to its neighbors

interface <set/remove> name <interface name> address <ipaddress/len> vrf <vrf name>

For example:
./agent_cli -p 50056 interface set name INT2 address 78.1.1.1/24 vrf vrf1
./agent_cli -p 50056 interface set name INT address 10.1.1.1/24 vrf global

## Sample configs
Configs directory has sample GoBGP configs.
l3vpn-setup.md describes a basic L3VPN setup with two VPP instances, two GoBGPds and two FIB agents

## Grpc 
This directory contains protobuf file for FIB agent and GoBGP.
This proto file can be compiled gRPC RPC APIs

## Some important steps/sequences:

As of today, fibagent needs to started first, before gobgpd is started.
GoBGPd tries to make a gRPC dial out towards the fibagent gRPC server. If gRPC server/fibagent is not started, gobgpd exits. 

## Important Notes

*We are still developing and testing many part of the code.
So, there could be issues in the binaries and functionalities.
Please report the issues back to us and we will address them in future updates.
email: go-kairos@cisco.com*
