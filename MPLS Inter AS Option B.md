# Objective
The objective is to set up L3VPN support that spans different autonomous systems to serve customers with sites on different service provider domains. In this case with reference to below topology, customers of AS64512 require to be connected to their other sites which are in AS64515.

<img width="720" alt="Inter AS Option B" src="https://user-images.githubusercontent.com/50369643/62412122-af61ec00-b606-11e9-9398-9a0626086d80.png">  
  
# Design
The design implemented in this case is known as Inter AS Option B in which the ASBRs will be exchanging MPLS labels via BGP as more than often two different AS will not be running LDP or RSVP but BGP. This means we also have to make the ASBR accept routing updates for VRFs as by default a router will only accepts VRF routes if itself is a member of that VRF and the ASBR may not be a member of all customers VRFs.
The ASBRs will accept VRF routes from the route reflectors and exchange that information with each other making VRF routes from AS64512 available in AS64515 and vice versa. In this design case AS64512 is an all Cisco gear networks while AS64515 is an all Juniper network both ASs use OSPF as IGP and LDP for label distribution both protocols are preconfigured.
With this design the route targets for a particular VRF are required to match on both Ass but you may run in a situation where the route targets in one AS are already being used by another customer in this case you can modify the route target community as the ASBR exchange the route information.

## Configuration
### ASBR64515
<pre>
protocols {
    mpls {
        interface em2.5;
        interface em1.200;
    }
    bgp {
        group iBGP {
            type internal;
            local-address 10.200.10.33;
            family inet {
                unicast;
            }
            family inet-vpn {
                unicast;
            }
            export POL_iBGP_EXPORT;
            peer-as 64515;
            local-as 64515;             
            neighbor 10.200.10.29;
        }
        group eBGP {
            type external;
            family inet {
                labeled-unicast;
            }
            family inet-vpn {
                unicast;
            }
            export POL_eBGP_EXPORT;
            peer-as 64512;
            local-as 64515;
            neighbor 10.100.100.2;
        }
    }
    ospf {
        area 0.0.0.0 {
            interface em2.5;
            interface lo0.400 {
                passive;
            }
        }                               
    }
    ldp {
        interface em2.5;
        interface lo0.400;
    }
}
policy-options {
    policy-statement POL_eBGP_EXPORT {
        term SWAP_RT_VRF_BBB {
            from community BBB200;
            then {
                community set BBB100;
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
    community BBB100 members target:100:2;
    community BBB200 members target:200:2;
}
routing-options {
    router-id 10.200.10.33;
    autonomous-system 64515;
}
</pre>

### ASBR64512
<pre>
router bgp 64512
neighbor iBGP-RR peer-group
 neighbor iBGP-RR remote-as 64512
 neighbor iBGP-RR update-source Loopback0
 neighbor eBGP peer-group
 neighbor eBGP remote-as 64515
 <b>neighbor eBGP send-label</b>
 neighbor 10.100.10.29 peer-group iBGP-RR
 neighbor 10.100.100.1 peer-group eBGP
 !
 address-family vpnv4
  <b>no bgp default route-target filter</b>
  neighbor iBGP-RR send-community both
  neighbor iBGP-RR next-hop-self
  neighbor eBGP send-community both
  neighbor eBGP next-hop-self
  neighbor eBGP route-map MAP_INTER_AS_NNI out
  neighbor 10.100.10.29 activate
  neighbor 10.100.100.1 activate
 exit-address-family
!
ip extcommunity-list standard RT100:2 permit rt 100:2
!
route-map MAP_INTER_AS_NNI permit 10
 description SWAP RT FOR VRF BBB
 match extcommunity SWAP_RT100:2_RT200:2 RT100:2
 set extcommunity rt 200:2
route-map MAP_INTER_AS_NNI permit 20
 description PERMIT OTHERS
!
interface FastEthernet0/0.200
 description NNI TO AS64515
 encapsulation dot1Q 200
 ip address 10.100.100.2 255.255.255.0
 <b>mpls bgp forwarding</b>

</pre>

The route reflectors and other PEs are configured with standard MPLS/IP support.

## Verification
### ASBR64515
<pre>
show mpls interface logical-system OPT-B  
Interface        State       Administrative groups (x: extended)
em1.200          Up         <none>
em2.5            Up         <none>
show bgp neighbor 10.100.100.2 | match NLRI
  NLRI for restart configured on peer: inet-vpn-unicast inet-labeled-unicast
  NLRI advertised by peer: inet-unicast inet-vpn-unicast inet-labeled-unicast
  <b>NLRI for this session: inet-vpn-unicast inet-labeled-unicast</b>

show bgp summary | find ^peer    
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.100.100.2          64512        251        252       0       2       15:20 Establ
  inet.0: 0/0/0/0
  <b>bgp.l3vpn.0: 2/2/2/0</b>

show route table bgp.l3vpn.0 advertising-protocol bgp 10.100.100.2
bgp.l3vpn.0: 4 destinations, 4 routes (4 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  100:1:10.40.0.36/32                    
*                         Self                                    I
  200:2:172.16.0.0/24                    
*                         Self                                    I

</pre>

### ASBR64512  
<pre>
sh mpls interface
Interface              IP            Tunnel   BGP Static Operational
FastEthernet0/1        Yes (ldp)     No       No  No     Yes        
FastEthernet0/0.200    No            No       Yes No     Yes  

sh bgp vpnv4 unicast all neighbors 10.100.100.1
  Neighbor capabilities:
    Route refresh: advertised and received(new)
    Four-octets ASN Capability: advertised and received
    Address family IPv4 Unicast: advertised
    <b>ipv4 MPLS Label capability: advertised and received</b>
    <b>Address family VPNv4 Unicast: advertised and received</b>
    Graceful Restart Capability: received
      Remote Restart timer is 120 seconds
      Address families advertised by peer:
        none
    Enhanced Refresh Capability: advertised
    Multisession Capability: 
    Stateful switchover support enabled: NO for session 1  

sh bgp vpnv4 unicast all peer-group eBGP summary | begin Neighbor
Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ Up/Down  State/PfxRcd
10.100.100.1    4        64515      34      36       19    0    0 00:13:43        <b>2</b>

sh bgp vpnv4 unicast all neighbor 10.100.100.1  advertised-routes | begin Network      
     Network          Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 100:1
 *>i 192.168.10.0/30  10.100.10.36             0    100      0 ?
Route Distinguisher: 100:2
 *>i 192.168.168.0/30 10.100.10.36             0    100      0 ?

Total number of prefixes 2
</pre>

We can see from above show commands both ASBRs have negotiated MPLS label capabilities and are announcing VRF routes to each other. The routes will then be advertised to the route reflectors and to the rest of the PEs in each AS



