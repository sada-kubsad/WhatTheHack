# Challenge 04 - Introduce High Availability with Central Network Virtual Appliances - Coach's Guide 

[< Previous Solution](./Solution-03.md) - **[Home](./README.md)**
           
## Notes & Guidance

- First instance of Central NVA has been deployed in challange 01. 
- Deploy another instance for the purpose of High Availability.
- Use same "Hub" Azure Virtual network but add two new subnets.
- Establish BGP Peering with new NVA using cheat sheet made available in the challange 02 section.
- Setup Internal Load Balancer.
- Update Route advertisements as necessary. 
- Internal Load Balancers are required for traffic symmetry.

# 1. Deploy another instance Central NVA for HA
</br>Use existing "Hub" Virtual network and existing subnets
```
# Variables
#Resource group name where the VPN VNG is located:
rg=wthars

#Cisco Image
publisher=cisco
offer=cisco-c8000v-byol
sku=17_13_01a-byol
version=latest

#Credentials
ADMIN_PASSWORD=Msft123Msft123
username=azureuser

# NVA goes in this subnet 
nva_subnet_name=nva
nva_subnet_prefix='10.0.1.0/24'


# Try to find a VNG in the RG
echo "Searching for a VPN gateway in the resource group $rg..."
vpngw_name=$(az network vnet-gateway list -g "$rg" --query '[0].name' -o tsv)
location=$(az network vnet-gateway show -n "$vpngw_name" -g "$rg" --query location -o tsv)
vnet_name=$(az network vnet-gateway show -n "$vpngw_name" -g "$rg" --query 'ipConfigurations[0].subnet.id' -o tsv | cut -d/ -f 9)
# Create NVA
nva_name="${vnet_name}-nva2"
nva_pip_name="${vnet_name}-nva2-pip"
version=$(az vm image list -p $publisher -f $offer -s $sku --all --query '[0].version' -o tsv)
echo "Creating NVA $nva_name in VNet $vnet_name in location $location from ${publisher}:${offer}:${sku}:${version}, in resource group $rg..."
az vm create -n "$nva_name" -g "$rg" -l "$location" \
    --image "${publisher}:${offer}:${sku}:${version}" \
    --admin-username "$username" --admin-password "$ADMIN_PASSWORD" --authentication-type all --generate-ssh-keys \
    --public-ip-address "$nva_pip_name" --public-ip-address-allocation static \
    --vnet-name "$vnet_name" --subnet "$nva_subnet_name" --subnet-address-prefix "$nva_subnet_prefix" -o none --only-show-errors
# Configuring the NIC for IP forwarding
nva_nic_id=$(az vm show -n $nva_name -g $rg --query 'networkProfile.networkInterfaces[0].id' -o tsv)
az network nic update --ids $nva_nic_id --ip-forwarding -o none --only-show-errors
```
#2. Establish BGP Peering between NVA-2 and Azure Route Server
## 2.1 Configure ARS for BGP
[Configure ARS](https://learn.microsoft.com/en-us/azure/route-server/quickstart-configure-route-server-portal#set-up-peering-with-nva) with Peer ASN = 65001 and Peer (NVA) IP Address: 10.0.1.4
Important Note:

ARS ASN 65515
NVA ASN 65001
VPN gateway ASN 65515 (default value)
On-Prem CSR ASN 65501

## 2.2 Configure BGP on Central Hub CSR - 2:
[Based on this guide](https://github.com/sada-kubsad/WhatTheHack/blob/master/057-AzureRouteServer/Student/Resources/whatthehackcentralnvachallenge2.md#sample-deployment-script)
```
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
#3. Setup Internal Load Balancer
</br>Internal Load Balancers are required for traffic symmetry.
```
```
#4. Update Route advertisements as necessary
```
```
