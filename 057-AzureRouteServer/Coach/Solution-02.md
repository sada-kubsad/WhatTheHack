# Challenge 02 - Introduce Azure Route Server and peer with a NVA - Coach's Guide 

[< Previous Solution](./Solution-01.md) - **[Home](./README.md)** - [Next Solution >](./Solution-03.md)

## Notes & Guidance

# 1. Undo/remove static route tabls (UDRs) <br/>
- You can't remove branch UDR since it's an Azure VNet simulating on-premises (running on Azure SDN) <br/>
- This stays:
``` bash
    az network route-table create -g $rg -n BranchVMSubnetToHubSpokeVNet
    az network vnet subnet update -g $rg --vnet-name datacenter -n vm --route-table BranchVMSubnetToHubSpokeVNet
    az network route-table route create -g $rg --route-table-name BranchVMSubnetToHubSpokeVNet -n           BranchVMSubnetToHubSpokeVNet --address-prefix 10.0.0.0/8 --next-hop-type VirtualAppliance  --next-hop-ip-address 172.16.1.10
```
# 2. Deploy Azure Route Server
You can use this script to deploy Azure Route Server. Setting up via portal is recommended if you are new to Azure Route Server. 

```bash

# Create Azure Route Server

echo "Creating Azure Route Server"
az network vnet subnet create --name RouteServerSubnet --resource-group $rg --vnet-name $vnet_name --address-prefix 10.0.3.0/24


subnet_id=$(az network vnet subnet show --name RouteServerSubnet --resource-group $rg --vnet-name $vnet_name --query id -o tsv) 
echo $subnet_id

az network public-ip create --name RouteServerIP --resource-group $rg --version IPv4 --sku Standard

az network routeserver create --name ARSHack --resource-group $rg --hosted-subnet $subnet_id --public-ip-address RouteServerIP

# Enable Branch to Branch flag.
az network routeserver update --name ARSHack --resource-group $rg --allow-b2b-traffic true
```

# 3. Setup BGP peering with Central NVA <br/>
## 3.1 Configure ARS to BGP peer with NVA
[Configure ARS](https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva) with Peer ASN = 65001 and Peer IP Address: 10.0.1.4</br>
Note: 
- ARS ASN 65515.
- NVA ASN 65001
- VPN gateway ASN 65515 by default 

## 3.2 Configure CSR:
[Based on this guide](https://github.com/sada-kubsad/WhatTheHack/blob/master/057-AzureRouteServer/Student/Resources/whatthehackcentralnvachallenge2.md#sample-deployment-script)
``` bash
conf t
 router bgp 65001
 bgp log-neighbor-changes
 neighbor 10.0.3.4 remote-as 65515
 neighbor 10.0.3.4 ebgp-multihop 255
 neighbor 10.0.3.4 update-source GigabitEthernet1
 neighbor 10.0.3.5 remote-as 65515
 neighbor 10.0.3.5 ebgp-multihop 255
 neighbor 10.0.3.5 update-source GigabitEthernet1
exit
exit
wr mem
```
** Note:
router bgp 65515 <- ASN on NVA cannot be 65515. Will result in "Error: This BGP peer cannot share the same ASN as the virtual hub." when configuring Peers section of the ARS config. 
- ASN of NVA: 65001
- ASN of ARS: 65515
See [here](https://blog.cloudtrooper.net/2021/03/08/connecting-your-nvas-to-expressroute-with-azure-route-server/)

**
# 4 Validate configurations
## 4.1 ARS is actually comprised of 2 different instances, each with its own IP Address:
```bash
az network routeserver show --name ARSHack  --query virtualRouterIps
[
  "10.0.3.4",
  "10.0.3.5"
]
```
Returns  "10.0.3.4", "10.0.3.5"

## 4.2 Show BGP neighbours of VPN Gateway
```bash
az network vnet-gateway list-bgp-peer-status  -g wthars -n vpngw -o table

Neighbor     ASN    State      ConnectedDuration    RoutesReceived    MessagesSent    MessagesReceived
-----------  -----  ---------  -------------------  ----------------  --------------  ------------------
172.16.1.10  65001  Connected  02:40:19.5787142     1                 263             223
10.0.0.5     65015  Unknown                         0                 0               0
10.0.0.4     65015  Connected  4.18:20:10.6427316   2                 8468            7922
10.0.3.4     65015  Connected  1.06:47:06.9716932   1                 2120            2125
10.0.3.5     65015  Connected  1.06:47:06.9716932   1                 2118            2124
172.16.1.10  65001  Connected  02:40:19.0221396     1                 192             188
10.0.0.5     65015  Connected  4.18:59:43.9347177   2                 7918            7927
10.0.0.4     65015  Unknown                         0                 0               0
10.0.3.4     65015  Connected  1.06:47:10.7889515   0                 2118            2120
10.0.3.5     65015  Connected  1.06:47:10.7889515   0                 2123            2123
```
**IMPORTANT Note: ARS (10.0.3.4 and 10.0.3.5) is now a BGP neighbour of the VPN Gateway (10.0.0.4 and 10.0.0.5) although ARS was never configured with VPN Gateway config. ARS was only configured with Centeral NVA BGP config**

- BGP Session is established between ARS’s instance IP (2 IPs) and NVA’s private IP. 
- BGP session is NOT established between ARS’s instance IP and VPN/ER Gateway’s BGP endpoint IPs (2 IPs). 
    - Diagrams often show a BGP session between ARS and VPN/ER Gateway although that physically not
- When ARS’s branch-to-branch traffic is enabled, ARS learns about the on-prem routes that come through the VPN/ER Gateway through the connection ARS has with Azure Routing not by establishing a connection to VPN/ER Gateway’s BGP endpoints IPs


## 4.3 Check that the Route Server is talking over BGP with the NVA at 10.0.1.4
```bash
az network routeserver peering list -g wthars --routeserver ARSHack -o table

Name           PeerAsn    PeerIp    ProvisioningState    ResourceGroup
-------------  ---------  --------  -------------------  ---------------
HubCentralNVA  65001      10.0.1.4  Succeeded            wthars
```
**See the important note above  that the ARS will NOT show the additional BGP peerings with the VPN gateways. It will only show the peering configured to the NVA, but nothing about the VPN gateways!**

## 4.4 ARS learned routes from the NVA
```bash
az network routeserver peering list-learned-routes --routeserver ARSHack -n HubCentralNVA -o table
```
Even though this may show nothing, that does not mean no routes have been sent by the NVA to teh ARS. See section below to check what routes the CSR is advertising to the NVA. 

## 4.5 ARS advertised routes  to the NVA
```bash
az network routeserver peering list-advertised-routes --routeserver ARSHack -n HubCentralNVA -o table
```
Even though this may show nothing, that does not mean no routes have been sent by the NVA to teh ARS. See section below to check what routes the CSR is learning from NVA. 

## 4.6 CSR advertised routes to NVA:
Show routes advertised to a particular neighbor

```bash
show ip bgp neighbors (neighbor ip) advertised-routes
```
neighbor ip:  10.0.3.4 and 10.0.3.5

## 4.7 CSR learned routes from NVA:
Show routes received from a particular neighbor
```bash
show ip bgp neighbors (neighbor ip) routes
```
neighbor ip:  10.0.3.4 and 10.0.3.5

## 4.8 VNET Gateway learned routes from NVA
```bash
az network vnet-gateway list-learned-routes -n vpngw -g $rg -o table
```

## 4.9 VNET Gateway advertised routes to NVA
```bash
az network vnet-gateway  list-advertised-routes -n vpngw -g $rg  --peer 10.0.3.4 -o table

Network        NextHop      Origin      SourcePeer    AsPath    Weight
-------------  -----------  ----------  ------------  --------  --------
172.16.1.0/24  172.16.1.10  Igp                       65001     0
172.18.0.0/16  172.16.1.10  Igp                       65001     0
172.16.1.0/26  172.16.1.10  Incomplete                65001     0
172.16.1.0/24  172.16.1.10  Igp                       65001     0
172.18.0.0/16  172.16.1.10  Igp                       65001     0
172.16.1.0/26  172.16.1.10  Incomplete                65001     0



az network vnet-gateway  list-advertised-routes -n vpngw -g $rg  --peer 10.0.3.5 -o table

Network        NextHop      Origin      SourcePeer    AsPath    Weight
-------------  -----------  ----------  ------------  --------  --------
172.16.1.0/24  172.16.1.10  Igp                       65001     0
172.18.0.0/16  172.16.1.10  Igp                       65001     0
172.16.1.0/26  172.16.1.10  Incomplete                65001     0
172.16.1.0/24  172.16.1.10  Igp                       65001     0
172.18.0.0/16  172.16.1.10  Igp                       65001     0
172.16.1.0/26  172.16.1.10  Incomplete                65001     0
```

## 4.10 VNET Gateway advertised routes to on-prem
```bash
az network vnet-gateway  list-advertised-routes -n vpngw -g $rg  --peer 172.16.1.10 -o table

Network      NextHop    Origin    SourcePeer    AsPath    Weight
-----------  ---------  --------  ------------  --------  --------
10.0.0.0/16  10.0.0.5   Igp                     65515     0
10.1.0.0/16  10.0.0.5   Igp                     65515     0
10.2.0.0/16  10.0.0.5   Igp                     65515     0
10.0.0.0/16  10.0.0.4   Igp                     65515     0
10.1.0.0/16  10.0.0.4   Igp                     65515     0
10.2.0.0/16  10.0.0.4   Igp                     65515     0
```

## 4.11 Routes on VM NICs:
```bash
az network nic show-effective-route-table --name hubvmVMNic -g wthars -o table
az network nic show-effective-route-table --name datacenter-nvaVMNic  -g wthars -o table
az network nic show-effective-route-table --name hub-nva1VMNic -g wthars -o table
az network nic show-effective-route-table --name spoke1-vmVMNic -g wthars -o table
az network nic show-effective-route-table --name spoke2-vmVMNic -g wthars -o table
az network nic show-effective-route-table --name onpremvm235_z1 -g wthars -o table
```

# 5. Test publishing routes/default routes on NVA<br/>
## 5.1 Advertise a default route
### 5.1.1 with default-originate
**On the Central Hub NVA execute:**
``` bash
conf t
!
router bgp 65001
address-family ipv4
neighbor 10.0.3.4 default-originate
neighbor 10.0.3.5 default-originate
end
```

### 5.1.2 with a static IP address
This can be very useful to advertise the range from the whole onprem environment (172.16.0.0/16) </br>

On the onprem nva aka datacenter NVA:  
```bash
conf t
!
ip route 172.16.1.0 255.255.255.0 10.0.1.4
!
router bgp 65001
 network 172.16.1.0 mask 255.255.255.0
end
```

## 5.2 Advertise a route with a loopback interface
This pattern can be used to advertise a route, and offer an IP address that is pingable to other systems. For example, if you want to simulate an SDWAN prefix 172.18.0.0, and Azure VMs should be able to ping the IP address 172.18.0.1:</br>

On the on-prem datacenter NVA: 
```bash
conf t
!
interface Loopback0
  ip address 172.18.0.1 255.255.0.0
  no shutdown
!
router bgp 65001
 network 172.18.0.0 mask 255.255.0.0
end
```

## 5.3 Show Static routes configured on a CSR:
```bash
show running | include ip route
```

## 5.4 Removing a static route on the CSR:
```bash
no ip route [destination network] [subnet mask]
```

# 6 Route manipulation
Some times you want to manipulate routes before advertising them to the Azure Route Server. 
## 6.1 Set a specific next hop
You can configure an outbound route map for the ARS neighbors that sets the next-hop field of the BGP route to a certain IP (typically an Azure Load Balancer in front of the NVAs):
```bash
conf t
router bgp 65001
  neighbor 10.0.3.4 route-map To-ARS out
  neighbor 10.0.3.5 route-map To-ARS out
route-map To-ARS
  set ip next-hop 10.0.1.200
end
```

## 6.2 Configure AS path prepending
AS path prepending is a technique that is frequently used to make certain routes less preferrable. To configure all routes advertised from an NVA to ARS with an additional ASN in the path:
```bash
conf t
router bgp 65001
  neighbor 10.0.3.4 route-map To-ARS out
  neighbor 10.0.3.5 route-map To-ARS out
route-map To-ARS
  set as-path prepend **NVA_ASN**
end
```

# 7. Validate traffic flows via NVA <br/>
## 7.1 Spoke to Spoke
- You will notice only spoke to spoke routing via NVA works <br/>
- For spoke to spoke traffic, advertise supernet since equal and more specific routes will be dropped by Azure Route Server because Azure already knows of the specific route.<br/>
- Also, because of the limitation in the Azure virtual network gateway type VPN it's not possible to advertise a 0.0.0.0/0 route through to the VPN Gateway. Student can try the approach of splitting advertisements like 0/1 and 128/1, but still routes learned through VPN Gateway will be more specific. (There might be some differences with ExR but same principle applies)<br/>

## 7.2 Azure to on-prem
- Azure to on-premises traffic is not possible in this design since advertising a more specific on-prem prefix from the NVA will result in a routing loop with the SDN fabric.

 ## 7.3 On-prem to Azure
- On-premises to Azure routing is possible but it requires static routes on Gateway and NVA Subnet (to prevent the loop)<br/>

Few plausible solutions to solve these challanges.
- [Create an overlay with Vxlan + eBGP](https://blog.cloudtrooper.net/2021/03/29/using-route-server-to-firewall-onprem-traffic-with-an-nva/) <br/>
- [Split ARS design](https://docs.microsoft.com/en-us/azure/route-server/route-injection-in-spokes#different-route-servers-to-advertise-routes-to-virtual-network-gateways-and-to-vnets)<br/>
