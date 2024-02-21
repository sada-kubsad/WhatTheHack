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
## 3.1 Configure ARS
[Configure ARS](https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva) with ASN = 65515 and IP Address: 10.0.1.4   

## 3.2 Configure CSR:
[Based on this guide](https://github.com/sada-kubsad/WhatTheHack/blob/master/057-AzureRouteServer/Student/Resources/whatthehackcentralnvachallenge2.md#sample-deployment-script)
``` bash
conf t
 router bgp 65501
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
- ASN of NVA: 65501
- ASN of ARS: 65515
See [here](https://blog.cloudtrooper.net/2021/03/06/route-server-multi-region-design/)

**
# 4. Test publishing routes/default routes on NVA<br/>
# 5. Validate traffic flows via NVA <br/>
## 5.1 Spoke to Spoke
- You will notice only spoke to spoke routing via NVA works <br/>
- For spoke to spoke traffic, advertise supernet since equal and more specific routes will be dropped by Azure Route Server because Azure already knows of the specific route.<br/>
- Also, because of the limitation in the Azure virtual network gateway type VPN it's not possible to advertise a 0.0.0.0/0 route through to the VPN Gateway. Student can try the approach of splitting advertisements like 0/1 and 128/1, but still routes learned through VPN Gateway will be more specific. (There might be some differences with ExR but same principle applies)<br/>

## 5.2 Azure to on-prem
- Azure to on-premises traffic is not possible in this design since advertising a more specific on-prem prefix from the NVA will result in a routing loop with the SDN fabric.

 ## 5.3 On-prem to Azure
- On-premises to Azure routing is possible but it requires static routes on Gateway and NVA Subnet (to prevent the loop)<br/>

Few plausible solutions to solve these challanges.
- [Create an overlay with Vxlan + eBGP](https://blog.cloudtrooper.net/2021/03/29/using-route-server-to-firewall-onprem-traffic-with-an-nva/) <br/>
- [Split ARS design](https://docs.microsoft.com/en-us/azure/route-server/route-injection-in-spokes#different-route-servers-to-advertise-routes-to-virtual-network-gateways-and-to-vnets)<br/>
