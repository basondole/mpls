# Inter AS MPLS VPN Option B for IPv6 VPNs (6VPE)
The objective is to set up an IPv6 L3VPN (6VPE) that spans different autonomous systems to serve customers with sites on different service provider domains. In this case with reference to below topology, customers of AS64512 require to be connected to their other sites which are in AS64515.

<img width="727" alt="6VPE Inter AS Option B" src="https://user-images.githubusercontent.com/50369643/62412342-9149bb00-b609-11e9-8216-a12584f2194b.png">

# Design
The design implemented in this case is known as Inter AS Option B in which the ASBRs will be exchanging MPLS labels via BGP as more than often two different AS will not be running LDP or RSVP but BGP. This means we also have to make the ASBR accept routing updates for VRFs as by default a router will only accepts VRF routes if itself is a member of that VRF and the ASBR may not be a member of all customers VRFs.
The ASBRs will accept VRF routes from the route reflectors and exchange that information with each other making VRF routes from AS64512 available in AS64515 and vice versa. In this design case AS64512 is an all Cisco gear networks while AS64515 is an all Juniper network both ASs use OSPF as IGP and LDP for label distribution both protocols are preconfigured.
With this design the route targets for a particular VRF are required to match on both Ass but you may run in a situation where the route targets in one AS are already being used by another customer in this case you can modify the route target community as the ASBR exchange the route information.

# Configuration
### ASBR64512
<pre>
interface Loopback0
 ip address 10.22.22.33 255.255.255.255
 ip ospf 1 area 0
!
interface FastEthernet0/0
 description INTER-AS LINK
interface FastEthernet0/0.200
 encapsulation dot1Q 200
 ip address 10.200.200.2 255.255.255.252
 mpls bgp forwarding
!
interface FastEthernet0/1
 description CONNECTION TO RR
 ip address 166.12.1.33 255.255.255.0
 ip ospf 1 area 0
mpls ip
!
router ospf 1
!
router bgp 200
 no bgp default route-target filter
 neighbor iBGP peer-group
 neighbor iBGP remote-as 200
 neighbor iBGP update-source Loopback0
 neighbor eBGP-6VPE peer-group
 neighbor eBGP-6VPE remote-as 400
 neighbor 10.22.22.29 peer-group iBGP
 neighbor 10.200.200.1 peer-group eBGP-6VPE
 !
 address-family ipv4
  neighbor iBGP next-hop-self
  neighbor eBGP-6VPE soft-reconfiguration inbound
  neighbor eBGP-6VPE send-label
  neighbor 10.22.22.29 activate
  neighbor 10.200.200.1 activate
 exit-address-family
 !
 address-family vpnv6
  neighbor iBGP send-community both
  neighbor iBGP next-hop-self
  neighbor eBGP-6VPE send-community both
  neighbor 10.22.22.29 activate
  neighbor 10.200.200.1 activate
 exit-address-family

### ASBR64515
interfaces {
    em2 {
        unit 5 {
            description "CONNECTION TO RR";
            vlan-id 5;
            family inet {
                address 172.17.1.33/24;
            }
            family mpls;
        }
    }
    em3 {
        unit 200 {
            description "NNI TO AS64512";
            vlan-id 200;
            family inet {
                address 10.200.200.1/30;
            }
            family mpls;
        }
    }
    lo0 {
        unit 400 {
            family inet {               
                address 10.44.44.33/32;
            }
        }
    }
}
protocols {
    mpls {
        <b>ipv6-tunneling;</b>
        interface em2.5;
        interface em3.200;
    }
    bgp {
        group iBGP {
            type internal;
            local-address 10.44.44.33;
            family inet {
                unicast;
            }
            family inet6-vpn {
                unicast;
            }
            export POL_iBGP_EXPORT;
            peer-as 64515;                
            local-as 64515;
            neighbor 10.44.44.29;
        }
        group eBGP {
            type external;
            </b>accept-remote-nexthop;</b>
            import POL_eBGP_IMPORT;
            family inet {
                <b>labeled-unicast;</b>
            }
            <b>family inet6-vpn {
                unicast;
            }</b>
            peer-as 64512;
            local-as 64515;
            neighbor 10.200.200.2;
        }
    }
    ospf {
        area 0.0.0.0 {
            interface lo0.400 {
                passive;
            }                           
            interface em2.5;
        }
    }
    ldp {
        interface em2.5;
        interface lo0.400;
    }
}
policy-options {
    policy-statement POL_eBGP_IMPORT {
        term CHANGE_VPNv6_NEXT_HOP {
            from family inet6-vpn;
            then {
                next-hop 10.200.200.2;
                accept;
            }
        }
    }
   policy-statement POL_iBGP_EXPORT {
        term NEXT_HOP {
            then {
                next-hop self;
            }
        }
    }
}
routing-options {
    router-id 10.44.44.33;              
    autonomous-system 64515;
}
</pre>

### RR64512
<pre>
interface Loopback0
 ip address 10.22.22.29 255.255.255.255
 ip ospf 1 area 0
!
interface FastEthernet0/0
 description CONNECTION TO PE
 ip address 172.16.0.29 255.255.255.0
 ip ospf 1 area 0
 mpls ip
!
interface FastEthernet0/1
 description CONNECTION TO ASBR
 ip address 172.16.1.29 255.255.255.0
 ip ospf 1 area 0
 mpls ip  
!
router ospf 1
!
router bgp 64512
 neighbor iBGP peer-group
 neighbor iBGP remote-as 200
 neighbor iBGP update-source Loopback0
 neighbor 10.22.22.33 peer-group iBGP
 neighbor 10.22.22.36 peer-group iBGP
 !
 address-family ipv4
  neighbor iBGP route-reflector-client
  neighbor 10.22.22.33 activate
  neighbor 10.22.22.36 activate
 exit-address-family
 !
 address-family vpnv6
  neighbor iBGP send-community both
  neighbor iBGP route-reflector-client
  neighbor 10.22.22.33 activate
  neighbor 10.22.22.36 activate
 exit-address-family
</pre>

### RR64515
<pre>
interfaces {
    em2 {
        unit 5 {
            description "CONNECTION TO ASBR-6VPE";
            vlan-id 5;
            family inet {
                address 172.17.1.29/24;
            }
            family inet6;
            family mpls;
        }
    }
    em3 {
        unit 10 {
            description "CONNECTION TO PE-6VPE";
            vlan-id 10;
            family inet {
                address 172.17.0.29/24;
            }
            family inet6;
            family mpls;
        }
    }
    lo0 {                               
        unit 400 {
            family inet {
                address 10.44.44.29/32;
            }
        }
    }
}
protocols {
    mpls {
        ipv6-tunneling;
        interface em2.5;
        interface em3.10;
    }
    bgp {
        group iBGP {
            type internal;
            local-address 10.44.44.29;
            family inet {
                unicast;
            }
            family inet6-vpn {
                unicast;
            }                           
            cluster 10.44.44.29;
            peer-as 64515;
            local-as 64515;
            neighbor 10.44.44.36;
            neighbor 10.44.44.33;
        }
    }
    ospf {
        area 0.0.0.0 {
            interface em2.5;
            interface lo0.400 {
                passive;
            }
            interface em3.10;
        }
    }
    ldp {
        interface em2.5;
        interface em3.10;
    }
}
routing-options {
    router-id 10.44.44.29;              
    autonomous-system 64515;
}
</pre>

### PE64512
<pre>
vrf definition AAA
 rd 200:1
 route-target export 200:1
 route-target import 200:1
 route-target import 100:1
 !
 address-family ipv4
 exit-address-family
 !
 address-family ipv6
 exit-address-family
!
ipv6 unicast-routing
ipv6 cef
!
interface Loopback0
 ip address 10.22.22.36 255.255.255.255
 ip ospf 1 area 0
!
interface FastEthernet0/1
 description 6VPE CUSTOMER
 vrf forwarding AAA
ipv6 address 2001:ABCD:200::36/128
!
interface FastEthernet0/0
 description CONNECTION TO RR
 ip address 172.16.0.36 255.255.255.0
 ip ospf 1 area 0
mpls ip
!
router ospf 1
!
router bgp 64512
no bgp default route-target filter
 neighbor iBGP peer-group
 neighbor iBGP remote-as 64512
 neighbor iBGP update-source Loopback0
 neighbor 10.22.22.29 peer-group iBGP
 !
 address-family ipv4
  neighbor iBGP next-hop-self
  neighbor 10.22.22.29 activate
 exit-address-family
 !
 address-family vpnv6
  neighbor iBGP send-community both
  neighbor iBGP next-hop-self
  neighbor 10.22.22.29 activate
 exit-address-family
 !
 address-family ipv6 vrf AAA
  redistribute connected
 exit-address-family
</pre>

### PE64515
<pre>
interfaces {
    em2 {
        unit 200 {
            description "6VPE CUSTOMER";
            vlan-id 200;
            family inet6 {
                address 2001:abcd:400::36/128;
            }
        }
    }
    em3 {
        unit 10 {
            description "CONNECTION TO RR";
            vlan-id 10;
            family inet {
                address 172.17.0.36/24;
            }
            family inet6;
            family mpls;
        }
    }
    lo0 {
        unit 400 {
            family inet {               
                address 10.44.44.36/32;
            }
        }
    }
}
protocols {
    mpls {
       <b> ipv6-tunneling;</b>
        interface em3.10;
    }
    bgp {
        group iBGP {
            type internal;
            local-address 10.44.44.36;
            family inet {
                unicast;
            }
            <b>family inet6-vpn {
                unicast;
            }</b>
            export POL_iBGP_EXPORT;
            peer-as 64515;
            local-as 64515;               
            neighbor 10.44.44.29;
        }
    }
    ospf {
        area 0.0.0.0 {
            interface lo0.400 {
                passive;
            }
            interface em3.10;
        }
    }
    ldp {
        interface em3.10;
        interface lo0.400;
    }
}
policy-options {
    policy-statement IMPORT_RT200_1 {
        from community RT_200_1;
        then accept;
    }
   policy-statement POL_iBGP_EXPORT {
        term NEXT_HOP {
            then {
                next-hop self;
            }
        }
    }
    community RT_200_1 members target:200:1;
}
routing-instances {
    AAA {
        instance-type vrf;
        interface em2.200;
        route-distinguisher 100:1;
        vrf-import IMPORT_RT200_1;
        vrf-target target:100:1;
        vrf-table-label;
    }
}
routing-options {
    router-id 10.44.44.36;
    autonomous-system 400;
}
</pre>

# Verification
In 6VPE BGP is used to distribute labels for FECs because LDP does not support IPv6 routes are announced with a IPv4-mapped IPv6 address as the next-hop, which is then resolved to an IPv4 address that will be next-hop.
IPv4-mapped IPv6 address format is ::FFFF::/96 the remaining 32 bits are derived from the corresponding IPv4 address

### ASBR64512
<pre>
sh bgp vpnv6 unicast all summary | b Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.22.22.29     4          64512      18      20       14    0    0 00:10:30        1
10.200.200.1    4          <b>64515</b>      33      30       14    0    0 00:11:56        <b>1</b>

#sh bgp vpnv6 unicast all | b Network
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 100:1
 *>  2001:ABCD:400::36/128
                       <b>::FFFF:10.200.200.1</b>
                                                              0 64515 i
Route Distinguisher: 200:1
 *>i 2001:ABCD:200::36/128
                       <b>::FFFF:10.22.22.36</b>
                                                0    100      0 ?
 *>  2001:ABCD:400::36/128
                       <b>::FFFF:10.200.200.1</b>
                                                              0 64515 i

</pre>

### ASBR64515
<pre>
run show bgp summary 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.44.44.29             64515         29         27       0       0       10:32 Establ
  inet.0: 0/0/0/0
  bgp.l3vpn-inet6.0: 1/1/1/0
10.200.200.2            <b>64512</b>         12         14       0       0        4:39 Establ
  inet.0: 0/0/0/0
  <b>bgp.l3vpn-inet6.0: 1/1/1/0</b>
</pre>

### RR64512
<pre>
sh bgp vpnv6 unicast all summary | b Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.22.22.33     4          64512      28      29        9    0    0 00:15:33        1
10.22.22.36     4          64512      15      22        9    0    0 00:10:11        1
</pre>

### RR64515
<pre>
run show bgp summary 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.44.44.33             64515        23         23       0       0        8:58 Establ
  inet.0: 0/0/0/0
  <b>bgp.l3vpn-inet6.0: 1/1/1/0</b>
10.44.44.36             64515         18         17       0       0        6:22 Establ
  inet.0: 0/0/0/0
  <b>bgp.l3vpn-inet6.0: 1/1/1/0</b>
</pre>

### PE64512
<pre>
sh bgp vpnv6 unicast all | b Network
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 100:1
 *>i 2001:ABCD:400::36/128
                       ::FFFF:22.22.22.33
                                                0    100      0 <b>64515 i</b>
Route Distinguisher: 200:1 (default for vrf AAA)
 *>  2001:ABCD:200::36/128
                       ::                       0         32768 ?
 *>i 2001:ABCD:400::36/128
                       ::FFFF:10.22.22.33
                                                0    100      0 <b>64515 i</b>
</pre>

### PE64515
<pre>
show bgp summary | find ^peer   
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.44.44.29             64515         13         13       0       0        4:18 Establ
  inet.0: 0/0/0/0
  <>bgp.l3vpn-inet6.0: 1/1/1/0</b>
  <b>AAA.inet6.0: 1/1/1/0</b>

show route table AAA.inet6.0 
AAA.inet6.0: 4 destinations, 5 routes (4 active, 0 holddown, 0 hidden)

2001:abcd:200::33/128
                   *[BGP/170] 00:00:51, MED 0, localpref 100, from 10.44.44.29
                      <b>AS path: 64512 ?</b>
                    > to 172.17.0.29 via em3.10, Push 299856, Push 299808(top)
2001:abcd:400::36/128
                   *[Direct/0] 00:02:13
                    > via em2.200
                    [Local/0] 00:02:13
                      Local via em2.200
fe80::/64          *[Direct/0] 00:02:13
                    > via em2.200
fe80::a00:2700:c864:3182/128
                   *[Local/0] 00:02:13
                      Local via em2.200
</pre>

Ping in VRF AAA from AS64512 to AS64515
<pre>
ping vrf AAA 2001:ABCD:400::36 source 2001:ABCD:200::36
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 2001:ABCD:400::36, timeout is 2 seconds:
Packet sent with a source address of 2001:ABCD:200::36%AAA
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 28/59/120 ms

traceroute vrf AAA 
Protocol [ip]: ipv6
Target IPv6 address: 2001:ABCD:400::36
Source address: 2001:ABCD:200::36
Timeout in seconds [3]: 1
Probe count [3]: 1
Type escape sequence to abort.
Tracing the route to 2001:ABCD:400::36

  1  * 
  2  * 
  3  * 
  4  * 
  5 2001:ABCD:400::36 [AS 64515] 56 msec
</pre>

Trace from AS64515 to AS64512
<pre>
traceroute routing-instance AAA 2001:abcd:200::36 source 2001:abcd:400::36
traceroute6 to 2001:abcd:200::36 (2001:abcd:200::36) from 2001:abcd:400::36, 64 hops max, 12 byte packets
 1  * * *
 2  * * *
 3  * * *
 4  * * *
 5  2001:abcd:200::36 (2001:abcd:200::36)  35.781 ms  60.482 ms  54.591 ms
</pre>
