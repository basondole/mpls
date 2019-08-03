# Traffic Engineering LSP Cisco X JunOS
Traffic engineering Label Switched Paths (LSPs) allows for granular control over what LSPs are used for MPLS traffic by analysing bandwidth available on different LSPs and choosing desirable paths based on resource  availability by using the Resource Reservation Protocol (RSVP) instead of the LDP. Generally these LSPs are unidirectional and require to be configured on both egress and ingress Label Switch Routers (LSR)

## Objective
In the network below it is required that traffic from END6 destined to END7 take the path PE1 to PE3 to PE5 to PE4 and PE2. Traffic from END7 destined to END6 should take the path PE2 to PE4 and PE1. VRF traffic from PE1 to PE2 should take the path PE1 to PE3 to PE5 to PE4 and PE2 while VRF traffic from PE2 to PE1 takes the path PE2 to PE5 to PE3 and PE1.  

<img width="714" alt="MPLSTE" src="https://user-images.githubusercontent.com/50369643/62410280-5b4a0e00-b5ec-11e9-999c-9a6254c15c3e.png">  

## Configuration

### PE1 config
<pre>
ip vrf L3VPN-TE
 rd 100:100
 route-target export 100:100
 route-target import 100:100
!
mpls traffic-eng tunnels
!         
interface Loopback0
 ip address 10.1.1.1 255.255.255.255
 ip ospf 1 area 0
!
interface Loopback100
 ip vrf forwarding L3VPN-TE
 ip address 10.1.1.1 255.255.255.255
!
interface Tunnel1
 description TE_TUNNEL_R1_R3_R4_R5_R2
 ip unnumbered Loopback0
 mpls traffic-eng tunnels
 tunnel mode mpls traffic-eng
 tunnel destination 10.2.2.2
 tunnel mpls traffic-eng autoroute announce
 tunnel mpls traffic-eng path-option 1 explicit name te-tunnel-R3-R4-R5-R2
!
interface Tunnel2
 description TE_TUNNEL_R1_R4_R3
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 10.3.3.3
 tunnel mpls traffic-eng forwarding-adjacency
 tunnel mpls traffic-eng priority 7 7
 tunnel mpls traffic-eng bandwidth 500
 tunnel mpls traffic-eng path-option 1 explicit name tunnel-R1-R4-R3
!
interface Tunnel3
 description TE_TUNNEL_R3_R5_R4_R2
 ip unnumbered Loopback0
 tunnel mode mpls traffic-eng
 tunnel destination 10.2.2.2
 tunnel mpls traffic-eng autoroute announce
 tunnel mpls traffic-eng path-option 1 explicit name tunnel-R3-R5-R4-R2
!         
interface FastEthernet0/0
 description connection to pe3
 ip address 10.13.31.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
interface FastEthernet1/0
 description connection to pe4
 ip address 10.14.41.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
interface FastEthernet2/0
 description connection to end6
 ip address 10.16.61.1 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 1
!
router ospf 1
 auto-cost reference-bandwidth 10
 mpls traffic-eng router-id Loopback0
 mpls traffic-eng area 0
!
router bgp 64512
 neighbor iBGP peer-group
 neighbor iBGP remote-as 64512
 neighbor iBGP update-source Loopback0
 neighbor 10.3.3.3 peer-group iBGP
 !
 address-family ipv4
  neighbor iBGP next-hop-self
  neighbor 10.3.3.3 activate
 exit-address-family
 !
 address-family vpnv4
  neighbor iBGP send-community extended
  neighbor iBGP next-hop-self
  neighbor 10.3.3.3 activate
 exit-address-family
 !
 address-family ipv4 vrf L3VPN-TE
  redistribute connected
 exit-address-family
!
ip explicit-path name te-tunnel-R3-R4-R5-R2 enable
 next-address 10.13.31.3
 next-address 10.34.43.4
 next-address 10.45.54.5
 next-address 10.25.52.2
!
ip explicit-path name tunnel-R1-R4-R3 enable
 next-address 10.14.41.4
 next-address 10.34.43.3
!
ip explicit-path name tunnel-R3-R5-R4-R2 enable
 next-address 10.13.31.3
 next-address 10.35.53.5
 next-address 10.45.54.4
 next-address 10.24.42.2
</pre>


### PE3 config
<pre>
mpls traffic-eng tunnels
!
interface Loopback0
 ip address 3.3.3.3 255.255.255.255
 ip ospf 1 area 0
!
interface FastEthernet0/0
 description connection to pe1
 ip address 10.13.31.3 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
interface FastEthernet1/0
 description connection to pe5
 ip address 10.35.53.3 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
interface FastEthernet2/0
 description connection to pe4
 ip address 155.34.43.3 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
interface FastEthernet3/0
 description connection to pe2
 ip address 10.23.32.3 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
router ospf 1
 auto-cost reference-bandwidth 10
 mpls traffic-eng router-id Loopback0
 mpls traffic-eng area 0
!
router bgp 64512
neighbor iBGP peer-group
 neighbor iBGP remote-as 64512
 neighbor iBGP update-source Loopback0
 neighbor 10.1.1.1 peer-group iBGP
 neighbor 10.2.2.2 peer-group iBGP
 neighbor 10.4.4.4 peer-group iBGP
 neighbor 10.5.5.5 peer-group iBGP
 !
 address-family ipv4
  neighbor iBGP route-reflector-client
  neighbor 10.1.1.1 activate
  neighbor 10.2.2.2 activate
  neighbor 10.4.4.4 activate
  neighbor 10.5.5.5 activate
 exit-address-family
 !
 address-family vpnv4
  neighbor iBGP send-community extended
  neighbor iBGP route-reflector-client
  neighbor 10.1.1.1 activate
  neighbor 10.2.2.2 activate
  neighbor 10.4.4.4 activate
  neighbor 10.5.5.5 activate
 exit-address-family
ip explicit-path name tunnel-R3-R4-R1 enable
 next-address 10.34.43.4
 next-address 10.14.41.1
</pre>


### PE4 config
<pre>
mpls traffic-eng tunnels
!
interface Loopback0
 ip address 10.4.4.4 255.255.255.255
 ip ospf 1 area 0
!
interface FastEthernet0/0
 description connection to pe5
 ip address 10.45.54.4 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 mpls ip
 mpls traffic-eng tunnels
!
interface FastEthernet1/0
 description connection to pe1
 ip address 10.14.41.4 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
interface FastEthernet2/0
 description connection to pe3
 ip address 10.34.43.4 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 mpls ip
 mpls traffic-eng tunnels
 ip rsvp bandwidth 100000
!
interface FastEthernet4/0
 description connection to pe2
 ip address 10.24.42.4 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 0
 mpls ip
 mpls traffic-eng tunnels
!
router ospf 1
 auto-cost reference-bandwidth 10
 mpls traffic-eng router-id Loopback0
 mpls traffic-eng area 0
!
router bgp 64512
 neighbor iBGP peer-group
 neighbor iBGP remote-as 64512
 neighbor iBGP update-source Loopback0
 neighbor 10.3.3.3 peer-group iBGP
 !
 address-family ipv4
  neighbor iBGP next-hop-self
  neighbor 10.3.3.3 activate
 exit-address-family
 !
 address-family vpnv4
  neighbor iBGP send-community extended
  neighbor iBGP next-hop-self
  neighbor 10.3.3.3 activate
 exit-address-family
</pre>


### PE5 config
<pre>
interfaces {
    em0 {
        unit 0 {
            description "connection to pe4";
            family inet {
                address 10.45.54.5/24;
            }
            family mpls;
        }
    }
    em1 {
        unit 0 {
            description "connection to pe3";
            family inet {
                address 10.35.53.5/24;
            }
            family mpls;
        }
    }
    em2 {
        unit 0 {
            description "connection to pe2";
            family inet {
                address 10.25.52.5/24;
            }
            family mpls;
        }                               
    }
    lo0 {
        unit 0 {
            family inet {
                address 10.5.5.5/32;
            }
        }
    }
}
routing-options {
    autonomous-system 64512;
}
protocols {
    rsvp {
        interface em0.0;
        interface em1.0;
        interface em2.0;
    }
    mpls {
        interface em0.0;
        interface em1.0;
        interface em2.0;
    }                                   
    bgp {
        group iBGP {
            type internal;
            local-address 10.5.5.5;
            family inet {
                unicast;
            }
            family inet-vpn {
                unicast;
            }
            export export-iBGP;
            peer-as 64512;
            neighbor 10.3.3.3 {
                description route-reflector;
            }
        }
    }
    ospf {
        traffic-engineering;
        area 0.0.0.0 {
            interface lo0.0 {
                passive;
            }                           
            interface em0.0 {
                interface-type p2p;
            }
            interface em1.0 {
                interface-type p2p;
            }
            interface em2.0 {
                interface-type p2p;
            }
        }
    }
    ldp {
        interface em0.0;
        interface em1.0;
        interface em2.0;
    }
}
policy-options {
    policy-statement export-iBGP {
        term next-hop {
            then {
                next-hop self;
                accept;                 
            }
        }
    }
}
</pre>


### PE2 config
<pre>
interfaces {
    em0 {
        unit 0 {
            description "connection to end7";
            family inet {
                address 10.27.72.2/24;
            }
        }
    }
    em2 {
        unit 0 {
            description "connection to pe5";
            family inet {
                address 10.25.52.2/24;
            }
            family mpls;
        }
    }
    em3 {
        unit 0 {
            description "connection to pe3";
            family inet {
                address 10.23.32.2/24;
            }
            family mpls;
        }
    }                                   
    em4 {
        unit 0 {
            description "connection to pe4";
            family inet {
                address 10.24.42.2/24;
            }
            family mpls;
        }
    }
    lo0 {
        unit 0 {
            family inet {
                address 10.2.2.2/32;
            }
        }
        unit 100 {
            family inet {
                address 10.2.2.2/32;
            }
        }
    }
}
routing-options {                       
    autonomous-system 64512;
}
protocols {
    rsvp {
        interface em2.0;
        interface em3.0 {
            bandwidth 100m;
        }
        interface em4.0;
    }
    mpls {
        label-switched-path tunnel-R2-R4-R1 {
            to 10.1.1.1;
            install 10.6.6.1/32 active;
            primary R2-R4-R1;
        }
        label-switched-path tunnel-R2-R5-R3-R1 {
            from 10.2.2.2;
            to 10.1.1.1;
            /* assigning prefixes to be reached via the LSP */
            install 10.6.6.0/32 active;
            preference 6;
            /* allocating an explicit-path */
            primary R2-R5-R3-R1;
        }
        path R2-R4-R1 {
            10.24.42.4;
            10.14.41.1;
        }
        path R2-R5-R3-R1 {
            10.25.52.5 strict;
            10.35.53.3 strict;
            10.13.31.1 strict;
        }
        interface em2.0;
        interface em3.0;
        interface em4.0;
    }
    bgp {
        group iBGP {
            type internal;
            local-address 10.2.2.2;
            family inet {
                unicast;
            }
            family inet-vpn {           
                unicast;
            }
            export export-iBGP;
            peer-as 64512;
            neighbor 10.3.3.3 {
                description route-reflector;
            }
        }
    }
    ospf {
        traffic-engineering;
        area 0.0.0.0 {
            interface lo0.0 {
                passive;
            }
            interface em2.0 {
                interface-type p2p;
                priority 1;
            }
            interface em4.0 {
                interface-type p2p;
            }
            interface em3.0 {           
                interface-type p2p;
            }
        }
        area 0.0.0.1 {
            interface em0.0 {
                interface-type p2p;
            }
        }
    }
    ldp {
        interface em2.0;
        interface em3.0;
        interface em4.0;
    }
}
policy-options {
    policy-statement export-iBGP {
        term nex-hop {
            then {
                next-hop self;
                accept;
            }
        }                               
    }
}
routing-instances {
    L3-VPN-TE {
        instance-type vrf;
        interface lo0.100;
        route-distinguisher 100:100;
        vrf-target target:100:100;
        vrf-table-label;
    }
}
</pre>


### END6 config
<pre>
interface Loopback0
 ip address 10.6.6.0 255.255.255.255
 ip ospf 1 area 1
!
interface Loopback1
 ip address 10.6.6.1 255.255.255.255
 ip ospf 1 area 1
!
interface Loopback2
 ip address 10.6.6.2 255.255.255.255
 ip ospf 1 area 1
!
interface Loopback3
 ip address 10.6.6.3 255.255.255.255
 ip ospf 1 area 1
!
interface FastEthernet2/0
 description connection to pe1
 ip address 10.16.61.6 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 1
!
router ospf 1
 auto-cost reference-bandwidth 10
</pre>


### END7 config
<pre>
interface Loopback0
 ip address 10.7.7.0 255.255.255.255
 ip ospf 1 area 1
!
interface Loopback1
 ip address 10.7.7.1 255.255.255.255
 ip ospf 1 area 1
!
interface Loopback2
 ip address 10.7.7.2 255.255.255.255
 ip ospf 1 area 1
!
interface Loopback3
 ip address 10.7.7.3 255.255.255.255
 ip ospf 1 area 1
!
interface FastEthernet0/0
 description connection to pe2
 ip address 10.27.72.6 255.255.255.0
 ip ospf network point-to-point
 ip ospf 1 area 1
!
router ospf 1
 auto-cost reference-bandwidth 10
</pre>

## Verification
### Routing tables
#### PE2
<pre>
root@pe2# run show route table inet.0 10.6/16 
inet.0: 28 destinations, 33 routes (28 active, 0 holddown, 0 hidden)
10.6.6.0/32         *[RSVP/6/1] 00:09:14, metric 3
                    <b>> to 10.25.52.5 via em2.0, label-switched-path tunnel-R2-R5-R3-R1</b>
                    [OSPF/10] 00:09:39, metric 4
                      to 10.23.32.3 via em3.0
                    > to 10.24.42.4 via em4.0
10.6.6.1/32         *[RSVP/7/1] 03:19:36, metric 3
                    <b>> to 10.24.42.4 via em4.0, label-switched-path tunnel-R2-R4-R1</b>
                    [OSPF/10] 00:09:39, metric 4
                      to 10.23.32.3 via em3.0
                    > to 10.24.42.4 via em4.0
10.6.6.2/32         *[OSPF/10] 00:09:39, metric 4
                      to 10.23.32.3 via em3.0
                    > to 10.24.42.4 via em4.0
10.6.6.3/32         *[OSPF/10] 00:09:39, metric 4
                      to 10.23.32.3 via em3.0
                    > to 10.24.42.4 via em4.0
</pre>  

### PE1
<pre>
pe1#sh ip route 10.7.0.0
Routing entry for 10.7.0.0/32, 4 known subnets
O IA     10.7.7.0 [110/4] via 10.2.2.2, 00:04:55, Tunnel1
                 [110/4] via 10.2.2.2, 00:04:55, Tunnel3
O IA     10.7.7.1 [110/4] via 10.2.2.2, 00:04:55, Tunnel1
                 [110/4] via 10.2.2.2, 00:04:55, Tunnel3
O IA     10.7.7.2 [110/4] via 10.2.2.2, 00:04:55, Tunnel1
                 [110/4] via 10.2.2.2, 00:04:55, Tunnel3
O IA     10.7.7.3 [110/4] via 10.2.2.2, 00:04:55, Tunnel1
                 [110/4] via 10.2.2.2, 00:04:55, Tunnel3
</pre>

### Traceroute from END6 to END7
<pre>
end6#trace 10.7.7.0
Type escape sequence to abort.
Tracing the route to 10.7.7.0
VRF info: (vrf in name/id, vrf out name/id)
  1 10.16.61.1 56 msec 48 msec 56 msec
  2 10.13.31.3 [MPLS: Label 40 Exp 0] 544 msec 144 msec * 
  3 10.35.53.5 [MPLS: Label 300288 Exp 0] 424 msec 160 msec * 
  4  * 
    10.45.54.4 [MPLS: Label 44 Exp 0] 176 msec 2312 msec
  5 10.24.42.2 88 msec 84 msec 68 msec
  6 10.27.72.6 88 msec 116 msec 88 msec
</pre>

### Trace from END7 to END6
<pre>
end7#trace 10.6.6.0
Type escape sequence to abort.
Tracing the route to 10.6.6.0
VRF info: (vrf in name/id, vrf out name/id)
  1 10.27.72.2 28 msec 28 msec 36 msec
  2 10.25.52.5 [MPLS: Label 300304 Exp 0] 28 msec 44 msec 28 msec
  3 10.35.53.3 [MPLS: Label 30 Exp 0] 68 msec 48 msec 72 msec
  4 10.13.31.1 72 msec 68 msec 44 msec
  5 10.16.61.6 68 msec 48 msec 68 msec
</pre>

10.6.6.2 is not directed to the LSP on PE2 and therefore will follow the IGP path

<pre>
end7#trace 10.6.6.2
Type escape sequence to abort.
Tracing the route to 10.6.6.2
VRF info: (vrf in name/id, vrf out name/id)
  1 10.27.72.2 36 msec 24 msec 28 msec
  2 10.24.42.4 48 msec 48 msec 48 msec
  3 10.14.41.1 [MPLS: Label 28 Exp 0] 92 msec 48 msec 52 msec
  4 10.16.61.6 76 msec 76 msec 72 msec
</pre>

## VRF Traffic  
### PE2 routing
<pre>
root@pe2# run show route table L3-VPN-TE 
L3-VPN-TE.inet.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
10.1.1.1/32         *[BGP/170] 00:13:45, MED 0, localpref 100, from 10.3.3.3
                      AS path: ?
                    > to 10.25.52.5 via em2.0, label-switched-path tunnel-R2-R5-R3-R1
10.2.2.2/32         *[Direct/0] 04:05:00
                    > via lo0.100

[edit]
root@pe2# run traceroute routing-instance L3-VPN-TE 10.1.1.1 
traceroute to 10.1.1.1 (10.1.1.1), 30 hops max, 40 byte packets
 1  10.25.52.5 (10.25.52.5)  0.400 ms  0.602 ms  0.957 ms
     MPLS Label=300304 CoS=0 TTL=1 S=0
     MPLS Label=23 CoS=0 TTL=1 S=1
 2  10.35.53.3 (10.35.53.3)  49.839 ms  20.097 ms  22.342 ms
     MPLS Label=30 CoS=0 TTL=1 S=0
     MPLS Label=23 CoS=0 TTL=1 S=1
 3  10.1.1.1 (10.1.1.1)  30.427 ms  19.988 ms  20.866 ms
</pre>


### PE1 routing 
<pre>
pe1#sh ip route vrf L3VPN-TE
Routing Table: L3VPN-TE
Gateway of last resort is not set

      10.0.0.0/32 is subnetted, 1 subnets
C        10.1.1.1 is directly connected, Loopback100
      10.0.0.0/32 is subnetted, 1 subnets
B        10.2.2.2 [200/0] via 10.2.2.2, 00:06:47

pe1#sh mpls forwarding-table 10.2.2.2 32
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
24    [T]  Pop Label  10.2.2.2/32       0             Tu1        point2point 
      [T]  Pop Label  10.2.2.2/32       0             Tu3        point2point 

[T]     Forwarding through a LSP tunnel.
        View additional labelling info with the 'detail' option
        
pe1#trace vrf L3VPN-TE 10.2.2.2
Type escape sequence to abort.
Tracing the route to 10.2.2.2
VRF info: (vrf in name/id, vrf out name/id)
  1 10.13.31.3 [MPLS: Labels 40/16 Exp 0] 104 msec 32 msec 36 msec
  2 10.34.43.4 [MPLS: Labels 45/16 Exp 0] 28 msec
    10.35.53.5 [MPLS: Labels 300288/16 Exp 0] 16 msec
    10.34.43.4 [MPLS: Labels 45/16 Exp 0] 24 msec
  3 10.45.54.4 [MPLS: Labels 44/16 Exp 0] 36 msec
    10.45.54.5 [MPLS: Labels 300272/16 Exp 0] 24 msec
    10.45.54.4 [MPLS: Labels 44/16 Exp 0] 20 msec
  4 10.2.2.2 20 msec 28 msec 36 msec
</pre>  

PE1 is load balancing between Tunnel1 and Tunnel3 both of terminate on PE2 with active LSPs from PE1.
