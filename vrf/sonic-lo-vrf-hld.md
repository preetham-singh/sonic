# SONiC Loopback per VRF support design spec draft

Table of Contents
<!-- TOC -->
<!-- /TOC -->

## Document History
| Version | Date       | Author       | Description                                      |
|---------|------------|--------------|--------------------------------------------------|
|   v.01  | 07/11/2019 | Preetham     | Per VRF Loopback support                         |

## Abbreviations

## Loopback pre VRF feature Requirement
1. Add/delete Loopback interfaces per VRF
2. Support to add IPv4 and IPv6 host address on these loopback interfaces
3. Support to use these loopback interfaces as source for various routing protocol control packet transmission. For instance in case of BGP multihop sessions source IP of the BGP packet will be outgoing interface IP which can change based on the interface status bringing down BGP session though there is alternate path to reach BGP neighbor. In case loopback interface is used as source BGP session would not flap.
4. These loopback interface IP address can be utilized for router-id generation on the VRF which could be utilized by routing protocols.
5. Support to use these interfaces as tunnel end points if required in future.
6. Support to use these interfaces as source for IP Unnumbered interface.

### Functional Description
Since Linux kernel supports only one Loopback per Net Name Space, VRF implementation using L3 Master Dev(L3MDEV) cannot get per VRF Loopback interface support.
To Meet requirements 1-4 above, below changes are proposed:
1. Implement Per VRF loopback interface support using dummy interface type since they are functionally identical to loopback interface.
2. Add/delete per VRF Loopback interfaces.
3. Add support to add/delete IPv4(/32) and IPv6(/128) address on per VRF Loopback interface.
4. Install these IPv4 and IPv6 address in hardware as IP2ME routes in corresponding VRF Table.

####Config DB:
There will be no change in config DB schema of Loopback interfaces.
```jason
"LOOPBACK_INTERFACE":{
    "Loopback0" : {},
    "Loopback1":{
        "vrf_name":"Vrf-yellow"
    },
    "Loopback2":{},
    "Loopback0|10.10.10.1/32": {},
    "Loopback1|11.11.11.1/32":{},
    "Loopback2|12.12.12.1/32":{}
}
```
Behavior difference:
1. Only Loopback0 interface IP will be applied no default system loopback interface - "lo"
2. For all non Loopback0 interfaces defined, we will be creating dedicated interfaces of type dummy in respective VRF.

## Agent Changes
### intfmgrd changes
- When interface add is received by intfmgrd for interface name starting with "Loopback" and suffix <1-999> intfmgrd will create netdevice in kernel with type dummy and enslave to respective VRF L3MDEV. Also add interface entry with VRF attribute to app-intf-table.
- When IP address add is received on these loopback interfaces, ip address will be applied on corresponding kernel loopback netdevice. Also add {interface_name:ip address} to app-intf-prefix-table.

### intforchd changes  
- When app-intf-table changes for loopback interface, intforch updates local cache of interface information with vrf binding information.
- When app-intf-prefix-table sees IP address add/delete for Loopback interface, vrf name is fetched from local interface cache to obtain VRF-ID and add IP2ME route with this VRF ID.

## CLI

```bash
// create loopback per VRF:
$ config loopback add Loopback<1-999> [<vrf_name>]

// delete loopback:
$ config loopback del Loopback<1-999>
```
