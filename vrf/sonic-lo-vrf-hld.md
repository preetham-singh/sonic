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

#### Config DB:
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

## User scenarios

Add Loopback interface to a VRF:
In case, user wants to configure Loopback say Loopback10 in Vrf-blue fllowing are the steps:
```bash
$ config loopback add Loopback10 Vrf-blue
```
This command does following operations:
- Create loopback interface entry in LOOPBACK_INTERFACE in config_db and vrf binding to Vrf-blue
- intfmgrd will create netdevice Loopback10 of type dummy and brings up the netdev.
This will result in below sequence of netdev events in kernel from intfmgrd
```bash
ip link add Loopback10 type dummy
ip link set dev Loopback10 up
ip link set Loopback10 master Vrf-blue
```
- intfsorch will store this interface-vrf binding in local cache of interface information


Add IP address on Loopback interface:
```bash
$ config interface ip add Loopback10 10.1.1.1/32
```
This command does following operations:
In intfmgr:
- When IP address add is received on these loopback interfaces, ip address will be applied on corresponding kernel loopback netdevice. Also add {interface_name:ip address} to app-intf-prefix-table.
In intforch
- When app-intf-prefix-table sees IP address add/delete for Loopback interface, vrf name is fetched from local interface cache to obtain VRF-ID and add IP2ME route with this VRF ID.


Delete Loopback interface:
```bash
$ config loopback del Loopback10
```
When user deletes loopback interface, first all IP configured on the interface will be removed from app-intf-prefix-table
Later interface itself will be deleted from INTERFACE table in config_db
In intfmgrd, this will flush all ip on netdev and deletes loopback netdev in kernel
Infrorchd will delete IP2ME routes from corresponding VRF and deletes local interface cache which holds vrf binding information.

### Pull Requests

```
- CLI changes are part of PR [581](https://github.com/Azure/sonic-utilities/pull/581)

- Interface template changes are part of PR [3171](https://github.com/Azure/sonic-buildimage/pull/3171)

- Intfmgrd & Intfsorch changes are made on VRF changes branch of Nephos: PR [5](https://github.com/tylerlinp/sonic-swss/pull/5)
```
