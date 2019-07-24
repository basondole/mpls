# Scenario
An existing client with many service has one interface dedicated (PE1 to CE1) carrying a mix of services vlan-ccc (for layer 2circuit) and routed IP services.  
The new requirement is that the client wants to have an additional layer 2 circuit to connect CE1 and new site CE3 in which they will be able to add/remove vlans as they wish without service provider intervention (this has to be achieved with no change to current setup). 

## Topology diagram
<img width="484" alt="QnQ" src="https://user-images.githubusercontent.com/50369643/61710540-874ad100-ad5a-11e9-9717-8dd383f8df76.png">


## Configuration
#### PE-1
```
root@PE-1> show configuration routing-instances VPLS 
instance-type vpls;
vlan-id all;
interface ge-0/0/1.5000;
protocols {
    vpls {
        no-tunnel-services;
        vpls-id 1000;
        mesh-group L2CIRC {
            neighbor 10.2.2.2 {
                ignore-encapsulation-mismatch;
            }
        }
    }
}
root@PE-1> show configuration interfaces ge-0/0/1          
description client-end;
flexible-vlan-tagging;
encapsulation flexible-ethernet-services;
unit 100 {
    description "Sample Existing IP Service";
    vlan-id 100;
    family inet {
        address 192.168.10.1/30;
    }
}
unit 300 {
    description "Sample Existing Martini L2vpn";
    encapsulation vlan-ccc;
    vlan-id 300;
    family ccc;
}
unit 5000 {
    description "New Service";
    encapsulation vlan-vpls;
    vlan-id-range 1001-2000;
    family vpls;
}
```

#### PE-3
```
root@PE-3> show configuration protocols l2circuit 
neighbor 10.1.1.1 {
    interface ge-0/0/2.1000 {
        virtual-circuit-id 1000;
        ignore-encapsulation-mismatch;
    }
}

root@PE-3> show configuration interfaces ge-0/0/2         
flexible-vlan-tagging;
encapsulation flexible-ethernet-services;
unit 1000 {
    encapsulation vlan-ccc;
    vlan-id 1000;
    input-vlan-map pop;
    output-vlan-map push;
}
```

#### GPON OLT
```
GPON-OLT#sh run | se interface
interface GigabitEthernet0/0
description ETHERNET CONNECTION TO PE-3
interface GigabitEthernet0/2
desciption PON CONNECTION TO ONT

GPON-OLT#sh vlan id 1000

VLAN Name          Status    Ports
---- ------------- --------- ---------------------------
1000 VLAN1000      active    Gi0/0, Gi0/2
```

#### MIKROTIK
```
[admin@MikroTik] >/interface ethernet
set [ find default-name=ether2 ] comment="CONNECTION TO ONU"
set [ find default-name=ether1 ] comment="CONNECTION TO CE"

[admin@MikroTik] >/interface vlan add interface=ether2 name=svlan1000 vlan-id=1000

[admin@MikroTik] >for vlan from 1001 to 2000 do={
/interface bridge add name=("vlan-" .$vlan. "-bridge")
/interface vlan add interface=ether1 name=("cvlan" .$vlan) vlan-id=$vlan
/interface vlan add interface=svlan1000 name=("vlan-" .$vlan. "-in-1000") vlan-id=$vlan
/interface bridge port add bridge=("vlan-" .$vlan. "-bridge") interface=("vlan-" .$vlan. "-in-1000")
/interface bridge port add bridge=("vlan-" .$vlan. "-bridge") interface=("cvlan" .$vlan)
}
```



## Verification

#### PE1
```
root@PE-1> show vpls connections | find "Instance: VPLS"  

Instance: VPLS
  VPLS-id: 1000
  Mesh-group connections: L2CIRC
    Neighbor              Type  St    Time last up          # Up trans
    10.2.2.2(vpls-id 1000)   rmt   Up   Jul 23 06:10:55 2019           1
      Remote PE: 10.2.2.2, Negotiated control-word: No
      Incoming label: 262145, Outgoing label: 299776
      Negotiated PW status TLV: No
      Localinterface:lsi.1048576, Status: Up, Encapsulation: ETHERNET
        Description: Intf - vpls VPLS neighbor 10.2.2.2 vpls-id 1000
      Flow Label Transmit: No, Flow Label Receive: No


root@PE-1> show vpls mac-table 

Routing instance : VPLS
 Bridging domain : __VPLS__, VLAN : 1001
   MAC                 MAC      Logical          NH     RTR
   address             flags    interface        Index  ID
   c2:02:7b:9f:00:00   D        ge-0/0/1.5000   
   c2:03:3a:f9:00:01   D        lsi.1048576     


Routing instance : VPLS
 Bridging domain : __VPLS__, VLAN : 1002
   MAC                 MAC      Logical          NH     RTR
   address             flags    interface        Index  ID
   c2:02:7b:9f:00:00   D        ge-0/0/1.5000   
   c2:03:3a:f9:00:01   D        lsi.1048576     
```

#### PE3
```
root@PE-3> show l2circuit connections interface ge-0/0/2.1000 | find "Neighbor: "
Neighbor: 10.1.1.1
    Interface                 Type  St     Time last up          # Up trans
    ge-0/0/2.1000(vc 1000)    rmt   Up     Jul 23 06:10:55 2019           1
      Remote PE: 10.1.1.1, Negotiated control-word: Yes (Null)
      Incoming label: 299776, Outgoing label: 262145
      Negotiated PW status TLV: No
      Local interface: ge-0/0/2.1000, Status: Up, Encapsulation: VLAN
```


#### GPON OLT
```
GPON-OLT #sh mac add dynamic vlan 1000
          Mac Address Table
-------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
1000    c202.7b9f.0000    DYNAMIC     Gi0/0
1000    c203.3af9.0001    DYNAMIC     Gi0/2
Total Mac Addresses for this criterion: 2
```

## Testing

### Router connected to CE1

```
router-1#sh run | se interface
interface FastEthernet0/0
 description CONNECTION TO CE1
 no ip address
interface FastEthernet0/0.100
 encapsulation dot1Q 100
 ip address 192.168.10.2 255.255.255.252
interface FastEthernet0/0.200
 encapsulation dot1Q 200
 ip address 192.168.1.1 255.255.255.0
interface FastEthernet0/0.1001
 encapsulation dot1Q 1001
 ip address 172.16.16.1 255.255.255.248
interface FastEthernet0/0.1002
 encapsulation dot1Q 1002
 ip address 10.10.10.1 255.255.255.248
interface FastEthernet0/0.1003
 encapsulation dot1Q 1003
 ip address 10.10.3.1 255.255.255.252
```

### Router connected to CE3

```
router-2#sh run | se interface
interface FastEthernet0/1
 description CONNECTION TO CE3
 no ip address
interface FastEthernet0/1.1001
 encapsulation dot1Q 1001
 ip address 172.16.16.2 255.255.255.248
interface FastEthernet0/1.1002
 encapsulation dot1Q 1002
 ip address 10.10.10.2 255.255.255.248
interface FastEthernet0/1.1003
 encapsulation dot1Q 1003
 ip address 10.10.3.2 255.255.255.252

```

#### Results of ping from router-1 to router-2 on different test vlans

```
router-1#sh arp
Protocol  Address      Age Hardware Addr   Type Interface
Internet  10.10.3.1    -  c202.7b9f.0000  ARPA  FastEthernet0/0.1003
Internet  10.10.3.2    52 c203.3af9.0001  ARPA  FastEthernet0/0.1003
Internet  10.10.10.1   -  c202.7b9f.0000  ARPA  FastEthernet0/0.1002
Internet  10.10.10.2   87 c203.3af9.0001  ARPA  FastEthernet0/0.1002
Internet  172.16.16.1  -  c202.7b9f.0000  ARPA  FastEthernet0/0.1001
Internet  172.16.16.2  37 c203.3af9.0001  ARPA  FastEthernet0/0.1001
Internet  192.168.1.1  -  c202.7b9f.0000  ARPA  FastEthernet0/0.200
Internet  192.168.10.2 -  c202.7b9f.0000  ARPA  FastEthernet0/0.100
```

```
router-2#sh arp
Protocol  Address    Age  Hardware Addr   Type   Interface
Internet  10.10.3.1   55  c202.7b9f.0000  ARPA  FastEthernet0/1.1003
Internet  10.10.3.2   -   c203.3af9.0001  ARPA  FastEthernet0/1.1003
Internet  10.10.10.1  90  c202.7b9f.0000  ARPA  FastEthernet0/1.1002
Internet  10.10.10.2  -   c203.3af9.0001  ARPA  FastEthernet0/1.1002
Internet  172.16.16.1 40  c202.7b9f.0000  ARPA  FastEthernet0/1.1001
Internet  172.16.16.2 -   c203.3af9.0001  ARPA  FastEthernet0/1.1001
Internet  192.168.1.2 -   c203.3af9.0000  ARPA  FastEthernet0/0.200
```

#### Packet capture of ping from router-2 to router-1 on internal vlan 1003

Packet has one tag as it goes from router-2 to Mikrotik

![pcap-r2-to-mk](https://user-images.githubusercontent.com/50369643/61712701-92543000-ad5f-11e9-914c-d6e08496ea9d.png)

Mikrotik assigns a seconds tag and sends the packet towards PE-3 with two vlan tags

![pcap-mk-to-pe3](https://user-images.githubusercontent.com/50369643/61712671-80728d00-ad5f-11e9-904a-f28406efb75d.png)



# Troubleshooting

To delete the cvlans added by the script in Mikrotik use below script

```
[admin@MikroTik] > for x from 1001 to 2000 do={
/int bridge port {remove [find bridge=("vlan-" .$x. "-bridge")]}
/int bridge {remove [find name=("vlan-" .$x. "-bridge")]}
/int vlan {remove [find name=("cvlan" .$x)]}
/int vlan {remove [find name=("vlan-" .$x. "-in-1000")]}
}
```
##### Note

To delete every bridge interface  
```[admin@MikroTik] >inter bri {remove [find]}```

To delete every vlan interface  
```[admin@MikroTik] >int vla {remove [find]}```
