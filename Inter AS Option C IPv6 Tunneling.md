# Inter AS MPLS VPN Option C for IPv6 (6PE)

## Objective   
The objective is to tunnel IPv6 (6PE) traffic that spans different autonomous systems. This implementation offers the ability to tunnel IPv6 across segments that do not support IPv6.
In this case with reference to below topology, the NNI between AS64512 and AS64515 does not support IPv6 yet both systems support IPv6 in their core network, therefore we need to tunnel IPv6 across the NNI which does not support IPv6 so as to bridge the IPv6 core networks from AS64512 and AS64515


<img width="727" alt="6PE Inter AS Option C" src="https://user-images.githubusercontent.com/50369643/62412530-8e9c9500-b60c-11e9-8473-9f11104817c8.png">


# Design
This design follows the principles of inter AS MPLS Option C in which ASBR64515 will redistribute the IP of RR64515 from OSPF to eBGP and redistribute to OSPF the IP of RR64512 received via eBGP from ASBR64512 as well as generate labels. ASBR64512 will redistribute the IP of RR64512 from OSPF to eBGP and redistribute to OSPF the IP of RR64515 received via eBGP from ASBR64515.
The RRs will exchange routing information with each other making IPv6 routes from AS64512 available in AS64515 and vice versa.

# Configuration
### PE64512
<pre>
interface Loopback0
 ip address 11.11.11.36 255.255.255.255
 ip ospf 1 area 0
!
interface FastEthernet0/0
 description CONNECTION TO RR
 ip address 155.12.0.36 255.255.255.0
 ip ospf 1 area 0
mpls ip
!
interface FastEthernet0/1
description SAMPLE CUSTOMER
ipv6 address 2001:ABCD:100::36/128
!
router ospf 1
!
router bgp 100
neighbor iBGP peer-group
 neighbor iBGP remote-as 100
 neighbor iBGP update-source Loopback0
 neighbor 10.11.11.29 peer-group iBGP
 !
 address-family ipv4
  neighbor iBGP next-hop-self
  neighbor 10.11.11.29 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  redistribute static
  neighbor iBGP send-community both
  neighbor iBGP next-hop-self
  neighbor iBGP send-label
  <b>neighbor 10.11.11.29 activate</b>
 exit-address-family
!
ip bgp-community new-format
!
ipv6 route 2002::/64 Null0
</pre>

### PE64515
<pre>
interfaces {
    em0 {
        unit 200 {
            description "SAMPLE CUSTOMER";
            vlan-id 200;
            family inet6 {
                address 2001:abcd:300::36/128;
            }
        }
    }
    em1 {
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
        unit 300 {
            family inet {               
                address 10.33.33.36/32;
            }
        }
    }
}
protocols {
    mpls {
        ipv6-tunneling;
        interface em1.10;
    }
    bgp {
        group iBGP {
            type internal;
            local-address 10.33.33.36;
            family inet {
                unicast;
            }
            <b>family inet6 {
                labeled-unicast {
                    explicit-null;
                }
            }</b>
            export POL_iBGP_EXPORT;     
            local-as 64515;
            neighbor 10.33.33.29;
        }
    }
    ospf {
        area 0.0.0.0 {
            interface lo0.300 {
                passive;
            }
            interface em1.10;
        }
    }
    ldp {
        interface em1.10;
        interface lo0.300;
    }
}
policy-options {
    policy-statement POL_iBGP_EXPORT {
        term NEXT_HOP {
            then {
                next-hop self;
            }                           
        }
        term IPV6_TUNNELLING {
            from family inet6;
            then accept;
        }
    }
}
routing-options {
    rib inet6.0 {
        static {
            route 2003::/64 discard;
        }
    }
    router-id 10.33.33.36;
    autonomous-system 64515;
}
</pre>

### RR64512
<pre>
interface Loopback0
 ip address 10.11.11.29 255.255.255.255
 ip ospf 1 area 0
!
interface FastEthernet0/0
 description CONNECTION TO PE1
 ip address 172.16.0.29 255.255.255.0
 ip ospf 1 area 0
mpls ip
!
interface FastEthernet0/1
 description CONNECTION TO ASBR1
 ip address 172.16.1.29 255.255.255.0
 ip ospf 1 area 0
mpls ip  
!
router ospf 1
!
router bgp 64512
 neighbor iBGP peer-group
 neighbor iBGP remote-as 100
 neighbor iBGP update-source Loopback0
 neighbor eBGP-6PE peer-group
 neighbor eBGP-6PE remote-as 64515
 neighbor eBGP-6PE ebgp-multihop 255
 neighbor eBGP-6PE update-source Loopback0
 neighbor 10.11.11.33 peer-group iBGP
 neighbor 10.11.11.36 peer-group iBGP
 neighbor 10.33.33.29 peer-group eBGP-6PE
 !
 address-family ipv4
  neighbor iBGP soft-reconfiguration inbound
  neighbor 10.11.11.33 activate
  neighbor 10.11.11.36 activate
  no neighbor 10.33.33.29 activate
 exit-address-family
 !        
 address-family ipv6
  neighbor iBGP send-community both
  neighbor iBGP route-reflector-client
  neighbor iBGP soft-reconfiguration inbound
  neighbor iBGP send-label
  neighbor eBGP-6PE send-label
  neighbor 10.11.11.33 activate
  neighbor 10.11.11.36 activate
  neighbor 10.33.33.29 activate
 exit-address-family
</pre>

### RR64515
<pre>
interfaces {
    em0 {
        unit 5 {
            description "CONNECTION TO ASBR";
            vlan-id 5;
            family inet {
                address 172.17.1.29/24;
            }
            family inet6;
            family mpls;
        }
    }
    em1 {
        unit 10 {
            description "CONNECTION TO PE";
            vlan-id 10;
            family inet {
                address 172.17.0.29/24;
            }
            family inet6;
            family mpls;
        }
    }
    lo0 {                               
        unit 300 {
            family inet {
                address 10.33.33.29/32;
            }
        }
    }
}
protocols {
    mpls {
        <b>ipv6-tunneling;</b>
        interface em0.5;
        interface em1.10;
    }
    bgp {
        group iBGP {
            type internal;
            local-address 10.33.33.29;
            family inet {
                unicast;
            }
            <b>family inet6 {
                labeled-unicast {
                    explicit-null;      
                }
            }</b>
            cluster 10.33.33.29;
            neighbor 10.33.33.36;
            neighbor 10.33.33.33;
        }
        group eBGP-6PE {
            type external;
            <b>multihop {
                ttl 255;
            }</b>
            local-address 10.33.33.29;
            <b>family inet6 {
                labeled-unicast {
                    explicit-null;
                }</b>
            }
            peer-as 64512;
            neighbor 10.11.11.29;
        }
    }
    ospf {
        area 0.0.0.0 {                  
            interface em0.5;
            interface lo0.300 {
                passive;
            }
            interface em1.10;
        }
    }
    ldp {
        interface em0.5;
        interface em1.10;
    }
}
routing-options {
    router-id 10.33.33.29;
    autonomous-system 64515;
}
</pre>

### ASBR64512
<pre>
interface Loopback0
 ip address 10.11.11.33 255.255.255.255
 ip ospf 1 area 0
!
interface Loopback100
 description SAMPLE CUSTOMER
ipv6 address 2001:ABCD:100::33/128
!
interface FastEthernet0/0
 description INTER-AS LINK
 no ip address
!
interface FastEthernet0/0.100
 encapsulation dot1Q 100
 ip address 10.100.100.1 255.255.255.252
 <b>mpls bgp forwarding</b>
!
interface FastEthernet0/1
 description CONNECTION TO RR
 ip address 172.16.1.33 255.255.255.0
 ip ospf 1 area 0
mpls ip
!
auto
!
router ospf 1
 <b>redistribute bgp 100 subnets route-map MAP_IGP_OUT</b>
!
router bgp  64512
 neighbor iBGP peer-group
 neighbor iBGP remote-as 100
 neighbor iBGP update-source Loopback0
 neighbor eBGP peer-group
 neighbor eBGP remote-as 300
 neighbor 10.11.11.29 peer-group iBGP
 neighbor 10.100.100.2 peer-group eBGP
 !
 address-family ipv4
  network 10.11.11.29 mask 255.255.255.255
  neighbor iBGP next-hop-self
  neighbor iBGP route-map MAP_iBGP_OUT out
  <b>neighbor eBGP send-label</b>
  neighbor 10.11.11.29 activate
  neighbor 10.100.100.2 activate
 exit-address-family
 !
 address-family ipv6
  neighbor iBGP send-community both
  neighbor iBGP next-hop-self
  neighbor iBGP send-label
  neighbor 10.11.11.29 activate
 exit-address-family
!
ip prefix-list AS_64515_RR seq 5 permit 10.33.33.29/32
!
route-map MAP_iBGP_OUT deny 10
 description DENY AS64515 RR IN iBGP
 match ip address prefix-list AS_64515_RR
route-map MAP_iBGP_OUT permit 20
 description PERMIT OTHERS
!
route-map MAP_IGP_OUT permit 10
 description ALLOW AS64515 RR IN IGP
 match ip address prefix-list AS_64515_RR
</pre>

### ASBR64515
<pre>
interfaces {
    em0 {
        unit 5 {
            description "CONNECTION TO RR";
            vlan-id 5;
            family inet {
                address 172.17.1.33/24;
            }
            family inet6;
            family mpls;
        }
    }
    em1 {
        unit 100 {
            description "NNI TO AS100";
            vlan-id 100;
            family inet {
                address 10.100.100.2/30;
            }
            family inet6;
            family mpls;
        }
    }
    lo0 {                               
        unit 300 {
            family inet {
                address 10.33.33.33/32;
            }
        }
    }
}
protocols {
    mpls {
        <b>ipv6-tunneling;</b>
        interface em0.5;
    }
    bgp {
        group iBGP {
            type internal;
            local-address10.33.33.33;
            family inet {
                unicast;
            }
            <b>family inet6 {
                labeled-unicast {
                    explicit-null;
                }</b>                       
            }
            export POL_iBGP_EXPORT;
            local-as 64515;
            neighbor 10.33.33.29;
        }
        group eBGP-6PE {
            type external;
            family inet {
                <b>labeled-unicast;</b>
            }
            export POL_eBGP_EXPORT;
            peer-as 64512;
            local-as 64515;
            neighbor 10.100.100.1;
        }
    }
    ospf {
        export POL_IGP_EXPORT;
        area 0.0.0.0 {
            interface lo0.300 {
                passive;
            }
            interface em0.5;            
        }
    }
    ldp {
        egress-policy LABELS;
        interface em0.5;
        interface lo0.300;
    }
}
policy-options {
    policy-statement LABELS {
        from {
            protocol [ bgp direct ];
            route-filter 0.0.0.0/0 prefix-length-range /32-/32;
        }
        then accept;
    }
    policy-statement POL_IGP_EXPORT {
        term AS64512_RR_IP {
            from {
                protocol bgp;
                route-filter 10.11.11.29/32 exact;
            }
            then accept;                
        }
    }
    policy-statement POL_eBGP_EXPORT {
        term AS64515_RR {
            from {
                protocol ospf;
                route-filter 10.33.33.29/32 exact;
            }
            then accept;
        }
    }
    policy-statement POL_iBGP_EXPORT {
        term NEXT_HOP {
            then {
                next-hop self;
            }
        }
        term AS64512_RR_IP {
            from {
                route-filter 10.11.11.29/32 exact;
            }
            then reject;
        }                               
    }
}
routing-options {
    router-id 10.33.33.33;
    autonomous-system 64515;
}
</pre>

## Verification
### ASBR64515
<pre>
show bgp summary | find ^peer 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.33.33.29             64515        172        163       0       0     1:11:45 Establ
  inet.0: 0/0/0/0
  inet6.0: 4/4/4/0
10.100.100.1            <b>64512</b>        165        173       0       1        3:27 Establ
  <b>inet.0: 1/1/1/0</b>
show route advertising-protocol bgp 10.100.100.1
inet.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.33.33.29/32          Self                 1                  I
</pre>

### ASBR64512
<pre>
sh bgp summary | b N
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.11.11.29     4          64512      81      94        5    0    0 01:13:03        0
10.100.100.2    4          <b>64515</b>      15      16        5    0    0 00:05:15        <b>1</b>

sh bgp neighbor 10.100.100.2 advertised-route | b Net
     Network          Next Hop            Metric LocPrf Weight Path
 *>  10.11.11.29/32   172.16.1.29              2         32768 i

Total number of prefixes 1
</pre>

### RR64512
<pre>
sh bgp * all summary     
For address family: IPv4 Unicast

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.11.11.33     4          64512     102      88        6    0    0 01:12:06        1
10.11.11.36     4          64512      92      89        6    0    0 01:11:08        0

For address family: IPv6 Unicast
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.11.11.33     4          64512     102      88       15    0    0 01:12:07        0
10.11.11.36     4          64512      92      89       15    0    0 01:11:08        2
10.33.33.29     4          <b>64515</b>      64      66       15    0    0 00:27:32        <b>2</b>

sh bgp ipv6 uni | b Net
     Network          Next Hop            Metric LocPrf Weight Path
 *>i 2001:ABCD:100::36/128
                       ::FFFF:10.11.11.36
                                                0    100      0 ?
 *>  <b>2001:ABCD:300::36/128
                       ::FFFF:10.33.33.29
                                                              0 64515 i</b>
 *>i 2002::/64        ::FFFF:10.11.11.36
                                                0    100      0 ?
 *>  <b>2003::/64        ::FFFF:10.33.33.29
                                                              0 64515 i</b>

sh mpls forwarding-table 10.33.33.29
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
17         17         10.33.33.29/32   976           Fa0/1      172.16.1.33
</pre>


### RR64515
<pre>
show bgp summary | find ^peer 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.11.11.29             64512         81         81       0       1       34:45 Establ
  <b>inet6.0: 2/2/2/0</b>
10.33.33.33             64512        198        206       0       0     1:27:51 Establ
  inet.0: 0/0/0/0
  inet6.0: 0/0/0/0
10.33.33.36             64512        200        197       0       0     1:25:56 Establ
  inet.0: 0/0/0/0
  inet6.0: 2/2/2/0

show route protocol bgp table inet6.0
inet6.0: 7 destinations, 8 routes (7 active, 0 holddown, 0 hidden)
2001:abcd:100::36/128
                   *[BGP/170] 00:37:06, localpref 100, from 10.11.11.29
                      AS path: 64512 ?
                    > to 172.17.1.33 via em0.5, Push 19, <b>Push 299904(top)</b>
2001:abcd:300::36/128
                   *[BGP/170] 00:56:08, localpref 100, from 10.33.33.36
                      AS path: I
                    > to 172.17.0.36 via em1.10, Push 2
2002::/64          *[BGP/170] 00:37:06, localpref 100, from 10.11.11.29
                      AS path: 64512 ?
                    > to 172.17.1.33 via em0.5, Push 20, <b>Push 299904(top)</b>
2003::/64          *[BGP/170] 00:56:09, localpref 100, from 10.33.33.36
                      AS path: I
                    > to 172.17.0.36 via em1.10, Push 2

show ldp path 10.11.11.29 
Output Session (label)          Input Session (label)
10.33.33.36:0(299824)           10.33.33.33:0<b>(299904)</b>
10.33.33.33:0(299824)
  Attached route:  10.11.11.29/32, Ingress route
</pre>

# Label analysis
<p>PE64512 announces 2001:ABCD: 100::36/128 to RR64512 which then advertises the prefix to RR64515 which then advertises the prefix to PE64515</p>

<pre>
PE64512#sh run int lo100
interface Loopback100
ipv6 address 2001:ABCD:100::36/128
end

PE64512#sh ipv6 route 2001:ABCD:100::36/128                
Routing entry for 2001:ABCD:100::36/128
  Known via <b>"connected"</b>, distance 0, metric 0, type receive, connected, bgp 64512
  Route count is 1/1, share count 0
  Routing paths:
    <b>receive via Loopback100</b>
      Last updated 01:49:12 ago

PE64512#sh bgp ipv6 unicast 2001:ABCD:100::36/128
BGP routing table entry for 2001:ABCD:100::36/128, version 2
Paths: (1 available, best #1, table default)
  Advertised to update-groups:
     1         
  Refresh Epoch 1
  Local
    :: from 0.0.0.0 (10.11.11.36)
      Origin incomplete, metric 0, localpref 100, weight 32768, valid, sourced, best
      <b>mpls labels in/out 20/nolabel</b>
      rx pathid: 0, tx pathid: 0x0

PE64512#show mpls forwarding-table labels <b>20</b>
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
20         <b>Pop Label</b>  2001:ABCD:100::36/128   \
                                       2876          <b>aggregate</b>

</pre>

<pre>
RR64512# sh bgp ipv6 uni 2001:ABCD:100::36/128  
BGP routing table entry for 2001:ABCD:100::36/128, version 3
Paths: (1 available, best #1, table default)
  Advertised to update-groups:
     1          5         
  Refresh Epoch 1
  Local, (Received from a RR-client), (received & used)
    ::FFFF:10.11.11.36 (metric 2) from 10.11.11.36 (10.11.11.36)
      Origin incomplete, metric 0, localpref 100, valid, internal, best
      <b>mpls labels in/out 19/20</b>
      rx pathid: 0, tx pathid: 0x0

RR64512#show mpls forwarding-table labels <b>19</b>
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
19         20         2001:ABCD:100::36/128   \
                                       3398          Fa0/0      172.16.0.36 
RR64512#show ip route 10.11.11.36
Routing entry for 10.11.11.36/32
  Known via "ospf 1", distance 110, metric 2, type intra area
  Last update from 172.16.0.36 on FastEthernet0/0, 01:43:26 ago
  Routing Descriptor Blocks:
  * 172.16.0.36, from 10.11.11.36, 01:43:26 ago, via FastEthernet0/0
      Route metric is 2, traffic share count is 1 

</pre>

<pre>
RR64515# run show route table inet6.0 2001:ABCD:100::36/128 extensive
inet6.0: 8 destinations, 9 routes (8 active, 0 holddown, 0 hidden)
2001:abcd:100::36/128 (1 entry, 1 announced)
TSI:
KRT in-kernel 2001:abcd:100::36/128 -> {indirect(131072)}
Page 0 idx 0 Type 1 val 9364cb0
    <b>Flags: Nexthop Change</b>
    Nexthop: Self
    Localpref: 100
    <b>AS path: [64512] 64515 ?</b>
    Communities:
Path 2001:abcd:100::36 from 11.11.11.29 Vector len 4.  Val: 0
        *BGP    Preference: 170/-101
                <b>Next hop type: Indirect</b>
                Address: 0x9334dd8
                Next-hop reference count: 3
                <b>Source: 10.11.11.29</b>
                Next hop type: Router, Next hop index: 696
                Next hop: 172.17.1.33 via em0.5, selected
                <b>Label operation: Push 19, Push 299904(top)</b>
                Label TTL action: prop-ttl, prop-ttl(top)
                <b>Protocol next hop: ::ffff:10.11.11.29</b>
                <b>Push 19</b>
                Indirect next hop: 94b01f8 131072
                State: <Active Ext>
                <b>Local AS:   64512 Peer AS:   64512</b>
                Age: 55:57      Metric2: 1 
                Task: BGP_100.10.11.11.29+35003
                Announcement bits (3): 0-KRT 1-BGP_RT_Background 2-Resolve tree 3 
                AS path: 100 ?
                Accepted
                Route Label: 19
                Localpref: 100
                Router ID: 10.11.11.29
                Indirect next hops: 1
                        <b>Protocol next hop: ::ffff:10.11.11.29 Metric: 1</b>
                        <b>Push 19</b>
                        Indirect next hop: 94b01f8 131072
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 172.17.1.33 via em0.5
                        ::ffff:10.11.11.29/128 Originating RIB: inet6.3
                          Metric: 1                       Node path count: 1
                          Forwarding nexthops: 1
                                Nexthop: 172.17.1.33 via em0.5

RR64515# run show route ::ffff:10.11.11.29 
inet6.3: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
::ffff:10.11.11.29/128
                   *[LDP/9] 01:03:02, metric 1
                    > to 172.17.1.33 via em0.5, <b>Push 299904</b>

RR64515# run show ldp path 10.11.11.29 
Output Session (label)          Input Session (label)
10.33.33.36:0(299824)           10.33.33.33:0<b>(299904)</b>
10.33.33.33:0(299824)
  Attached route:  10.11.11.29/32, Ingress route

</pre>

<pre>
PE64515# run show route table inet6.0 2001:ABCD:100::36/128 extensive
inet6.0: 8 destinations, 10 routes (8 active, 0 holddown, 0 hidden)
2001:abcd:100::36/128 (1 entry, 1 announced)
TSI:
KRT in-kernel 2001:abcd:100::36/128 -> {indirect(131070)}
        *BGP    Preference: 170/-101
                Next hop type: Indirect
                Address: 0x9334b08
                Next-hop reference count: 9
                <b>Source: 10.33.33.29</b>
                Next hop type: Router, Next hop index: 708
                Next hop: 172.17.0.29 via em1.10, selected
                <b>Label operation: Push 2</b>
                Label TTL action: prop-ttl
                <b>Protocol next hop: ::ffff:10.33.33.29</b>
                <b>Push 2</b>
                Indirect next hop: 948c000 131070
                State: <Active Int Ext>
                <b>Local AS:   64515 Peer AS:   64515</b>
                Age: 44:39      Metric2: 1 
                Task: BGP_300.10.33.33.29+179
                Announcement bits (2): 0-KRT 2-Resolve tree 3 
                AS path: 64515 ?
                Accepted                
                Route Label: 2
                Localpref: 100
                Router ID: 10.33.33.29
                Indirect next hops: 1
                        <b>Protocol next hop: ::ffff:10.33.33.29 Metric: 1</b>
                        <b>Push 2</b>
                        Indirect next hop: 948c000 131070
                        Indirect path forwarding next hops: 1
                                Next hop type: Router
                                Next hop: 172.17.0.29 via em1.10
                        <b>::ffff:10.33.33.29/128 Originating RIB: inet6.3</b>
                          Metric: 1                       Node path count: 1
                          Forwarding nexthops: 1
                                Nexthop: 172.17.0.29 via em1.10

# run show route ::ffff:10.33.33.29/128 
inet6.3: 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
::ffff:10.33.33.29/128
                   *[LDP/9] 01:46:54, metric 1
                    > to 172.17.0.29 via em1.10
</pre>

PE64515 Pushes label 2 because of the bgp protocol configuration
<pre>
    family inet6 {
        labeled-unicast {
            <b>explicit-null;</b>
        }
</pre>
