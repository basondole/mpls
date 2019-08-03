# Objective
The objective is to set up L3VPN that spans different autonomous systems to serve customers with sites on different service provider domains. In this case with reference to below topology, customers of AS64512 require to be connected to their other sites which are in AS64515.
  
  
<img width="720" alt="Inter AS Option C" src="https://user-images.githubusercontent.com/50369643/62412179-8e4dcb00-b607-11e9-89e3-3d642b00a260.png">

# Design
The design implemented in this case is known as Inter AS Option C in which the route reflectors from each AS will peer directly and exchange vpnv4 routes. For forwarding the ASBRs will be exchanging MPLS labels via BGP as more than often two different AS will not be running LDP or RSVP but BGP. 
ASBR64515 will redistribute the IP of RR64515 from OSPF to eBGP and redistribute to OSPF the IP of RR64512 received via eBGP from ASBR64512 as well as generate labels. ASBR64512 will redistribute the IP of RR64512 from OSPF to eBGP and redistribute to OSPF the IP of RR64515 received via eBGP from ASBR64515.
The RRs will exchange routing information with each other making VRF routes from AS64512 available in AS64515 and vice versa. In this design case AS64512 is an all Cisco gear networks while AS64515 is an all Juniper network both ASs use OSPF as IGP and LDP for label distribution both protocols are preconfigured.

# Configuration
## ASBR64512
<pre>
interface FastEthernet0/0.100
 encapsulation dot1Q 100
 ip address 10.100.100.2 255.255.255.252
 <b>mpls bgp forwarding</b>

router ospf 1
 router-id 10.11.11.33
 <b>redistribute bgp 64512 subnets route-map MAP_IGP_OUT</b>
!
router bgp 64512
 neighbor iBGP peer-group
 neighbor iBGP remote-as 64512
 neighbor iBGP update-source Loopback0
 neighbor eBGP peer-group
 neighbor eBGP remote-as 64515
 neighbor 10.11.11.29 peer-group iBGP
 neighbor 10.100.100.1 peer-group eBGP
 !
 address-family ipv4
  <b>redistribute ospf 1 route-map MAP_eBGP_OUT</b>
  neighbor iBGP next-hop-self
  neighbor iBGP soft-reconfiguration inbound
  neighbor iBGP route-map MAP_iBGP_OUT out
  neighbor eBGP soft-reconfiguration inbound
  <b>neighbor eBGP send-label</b>
  neighbor 10.11.11.29 activate
  neighbor 10.100.100.1 activate
 exit-address-family
 !
 address-family vpnv4
  neighbor iBGP send-community both
  neighbor iBGP next-hop-self
  neighbor 10.11.11.29 activate
 exit-address-family
!
ip prefix-list AS64515-RR seq 5 permit 10.33.33.29/32
!
ip prefix-list RR_IP seq 5 permit 10.11.11.29/32
!
route-map MAP_eBGP_OUT permit 10
 description ADVERTISE RR IP
 match ip address prefix-list RR_IP
!
route-map MAP_iBGP_OUT deny 10
 description DO NOT ADVERTISE RR IP FROM AS64515
 match ip address prefix-list AS64515-RR
route-map MAP_iBGP_OUT permit 20
 description PERMIT OTHERS
!
route-map MAP_IGP_OUT permit 10
 description ADVERTISE AS64515 RR IN IGP
 match ip address prefix-list AS64515-RR
</pre>

## RR64512
<pre>
router bgp 64512
neighbor iBGP peer-group
 neighbor iBGP remote-as 64512
 neighbor iBGP update-source Loopback0
 neighbor eBGP-MPLS peer-group
 neighbor eBGP-MPLS remote-as 64515
 <b>neighbor eBGP-MPLS ebgp-multihop 255</b>
 <b>neighbor eBGP-MPLS update-source Loopback0</b>
 neighbor 10.11.11.33 peer-group iBGP
 neighbor 10.11.11.36 peer-group iBGP
 neighbor 10.33.33.29 peer-group eBGP-MPLS
 !
 address-family ipv4
  neighbor iBGP route-reflector-client
  neighbor iBGP soft-reconfiguration inbound
  neighbor 10.11.11.33 activate
  neighbor 10.11.11.36 activate
  <b>no neighbor 10.33.33.29 activate</b>
 exit-address-family
 !
 address-family vpnv4
  neighbor iBGP send-community both
  neighbor iBGP route-reflector-client
  <b>neighbor eBGP-MPLS send-community both</b>
  neighbor 10.11.11.33 activate
  neighbor 10.11.11.36 activate
  <b>neighbor 10.33.33.29 activate</b>
 exit-address-family
</pre>

## ASBR64515
<pre>
protocols {
    mpls {
        interface em0.5;
        interface em3.100;
    }
    bgp {
        local-as 64515;
        group iBGP {
            type internal;
            local-address 10.33.33.33;
            family inet {
                unicast;
            }
            family inet-vpn {
                unicast;
            }
            export POL_iBGP_EXPORT;
            neighbor 10.33.33.29;       
        }
        group eBGP {
            type external;
            family inet {
                labeled-unicast;
            }
            export POL_eBGP_EXPORT;
            peer-as 64512;
            local-as 64515;
            neighbor 10.100.100.2;
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
        egress-policy POL_GENERATE_LABELS;
        interface em0.5;
        interface lo0.300;
    }
}
policy-options {
    policy-statement POL_GENERATE_LABELS {
        term GENERATE_LABEL_FOR_MY_LOOPBACK {
            from {
                protocol direct;
                route-filter 10.33.33.33/32 exact;
            }
            then accept;
        }
        term GENERATE_LABEL_FOR_AS64512_RR {
            from {
                protocol bgp;
                route-filter 10.11.11.29/32 exact;
            }
            then accept;
        }
    }
    policy-statement POL_IGP_EXPORT {
        term ADVERTISE_RR_IP_FROM_AS64512 {
            from {
                protocol bgp;
                route-filter 10.11.11.29/32 exact;
            }
            then accept;
        }
    }
    policy-statement POL_eBGP_EXPORT {
        term ADVERTISE_RR_LOOPBACK {
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
        term DENY_RR_IP_FROM_AS64512 {
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

## RR64515
<pre>
protocols {
    mpls {
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
            family inet-vpn {
                unicast;
            }
            cluster 10.33.33.29;
            neighbor 10.33.33.33;
            neighbor 10.33.33.36;       
        }
        group eBGP-MPLS {
            <b>multihop {
                ttl 255;
            }</b>
            local-address 10.33.33.29;
           <b> family inet-vpn {
                unicast;
            }</b>
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

The route reflectors and other PEs are configured with standard MPLS/IP support.

# Verification
## ASBR64515
<pre>
show mpls interface logical-system
Interface        State       Administrative groups (x: extended)
em0.5            Up         <none>
em3.100          Up         <none>

show bgp neighbor 10.100.100.2 | match NLRI
  NLRI for restart configured on peer: inet-labeled-unicast
  NLRI advertised by peer: inet-unicast inet-labeled-unicast
  <b>NLRI for this session: inet-labeled-unicast</b>

show route 10.11.11.29
inet.0: 10 destinations, 10 routes (10 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
10.11.11.29/32     *[BGP/170] 00:58:12, MED 2, localpref 100
                      AS path: 64512 ?
                    > to 10.100.100.2 via em3.100, Push 21

</pre>


## ASBR64512
<pre>
sh mpls interfaces 
Interface              IP            Tunnel   BGP Static Operational
FastEthernet0/1        Yes (ldp)     No       No  No     Yes        
FastEthernet0/0.100    No            No       Yes No     Yes        

sh bgp neighbor 10.100.100.1 | i ^BGP|v4
BGP neighbor is 10.100.100.1,  remote AS 64515, external link
    Address family IPv4 Unicast: advertised
    <b>ipv4 MPLS Label capability: advertised and received</b>

sh ip route 10.33.33.29
Routing entry for 10.33.33.29/32
  Known via "bgp 64512", distance 20, metric 1
  <b>Tag 64515, type external</b>
  <b>Redistributing via ospf 1</b>
  Advertised by ospf 1 subnets route-map MAP_IGP_OUT
  Last update from 10.100.100.1 01:03:54 ago
  Routing Descriptor Blocks:
  * 10.100.100.1, from 10.100.100.1, 01:03:54 ago
      Route metric is 1, traffic share count is 1
      AS Hops 1
      Route tag 64515
      MPLS label: 300656

</pre>

The ASBRs on both ASs are learning the IP address of the RR on the other AS via BGP and they are redistributing to their IGP

## RR64515
<pre>
show route 10.11.11.29    # route to RR in AS64512
<b>inet.0:</b> 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
10.11.11.29/32     *[OSPF/150] 01:27:59, metric 2, tag 0
                    > to 172.18.1.33 via em0.5

<b>inet.3:</b> 3 destinations, 3 routes (3 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both
10.11.11.29/32     *[LDP/9] 01:10:53, metric 1
                    > to 172.18.1.33 via em0.5, Push 300672

show bgp neighbor 10.11.11.29 | match "^Peer|NLRI"
Peer: 10.11.11.29+30637 <b>AS 64512</b> Local: 10.33.33.29+179 <b>AS 64515</b>
  NLRI for restart configured on peer: inet-vpn-unicast
  NLRI advertised by peer: inet-vpn-unicast
  <b>NLRI for this session: inet-vpn-unicast</b>

show bgp summary | find ^peer 
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.11.11.29           <b>64512</b>        103         94       0       3       41:35 Establ
  <b>bgp.l3vpn.0: 1/1/1/0</b>
show route table bgp.l3vpn.0 receive-protocol bgp 10.11.11.29
bgp.l3vpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  100:1:10.10.0.36/32                    
*                         10.11.11.29                             64512 ?

show route table bgp.l3vpn.0 advertising-protocol bgp 10.11.11.29
bgp.l3vpn.0: 2 destinations, 2 routes (2 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  100:1:10.30.0.36/32                    
*                         Self                                    I

</pre>

## RR64512
<pre>
sh ip route 10.33.33.29 # route to RR in AS64515
Routing entry for 10.33.33.29/32
  <b>Known via "ospf 1", distance 110, metric 1</b>
  Tag 64515, type extern 2, forward metric 1
  Last update from 172.16.1.33 on FastEthernet0/1, 01:17:55 ago
  Routing Descriptor Blocks:
  * 172.16.1.33, from 10.11.11.33, 01:17:55 ago, via FastEthernet0/1
      Route metric is 1, traffic share count is 1
      Route tag 64515

sh mpls forwarding-table 10.33.33.29 32
Local      Outgoing   Prefix           Bytes Label   Outgoing   Next Hop    
Label      Label      or Tunnel Id     Switched      interface              
16         16         10.33.33.29/32   1750          Fa0/1      172.16.1.33


sh bgp vpnv4 unicast all neighbor 10.33.33.29 | include ^BGP|v4         
<b>BGP neighbor is 10.33.33.29,  remote AS 64515, external link</b>
    <b>Address family VPNv4 Unicast: advertised and received</b>

sh bgp vpnv4 unicast all peer-group eBGP-MPLS summary | begin Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.33.33.29     4        <b>64515</b>      89      98       11    0    0 00:42:05        </b1>

sh bgp vpnv4 unicast all neighbor 10.33.33.29  advertised-routes | begin Network      
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 100:1
 *>i 10.10.0.36/32    10.11.11.36              0    100      0 ?

Total number of prefixes 1 
</pre>

The route reflectors on both ASs are learning about each other via IGP (assigning MPLS labels as well) and are exchanging VRF routes via the eBGP session.
The routes will then be advertised by the route reflectors to the rest of the PEs in each AS
