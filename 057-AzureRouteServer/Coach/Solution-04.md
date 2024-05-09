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
# Variables
rgname=wthars
LBName=NVALoadBalancer

# Create the load balancer resource
az network lb create --resource-group wthars --name nvaLoadBalancer --sku Standard --vnet-name hub --backend-pool-name myBackEndPool --frontend-ip-name myFrontEnd

# Create the health probe
 az network lb probe create \
    --resource-group $rgname \
    --lb-name $LBName \
    --name myHealthProbe \
    --protocol tcp \
    --port 80

# Create a load balancer rule
 az network lb rule create \
    --resource-group $rgname \
    --lb-name $LBName \
    --name myHTTPRule \
    --protocol tcp \
    --frontend-port 80 \
    --backend-port 80 \
    --frontend-ip-name myFrontEnd \
    --backend-pool-name myBackEndPool \
    --probe-name myHealthProbe \
    --idle-timeout 15 \
    --enable-tcp-reset true

# Create a network security group
az network nsg create \
    --resource-group $rgname \
    --name myNSG

# Create a network security group rule
az network nsg rule create \
    --resource-group $rgname \
    --nsg-name myNSG \
    --name myNSGRuleHTTP \
    --protocol '*' \
    --direction inbound \
    --source-address-prefix '*' \
    --source-port-range '*' \
    --destination-address-prefix '*' \
    --destination-port-range 80 \
    --access allow \
    --priority 200

# Add virtual machines to the backend pool
 BackendVMNICNameArray=(hub-nva1 hub-nva2)
  for vm in "${BackendVMNICNameArray[@]}"
  do
  az network nic ip-config address-pool add \
   --address-pool myBackendPool \
   --ip-config-name ipconfig$vm \
   --nic-name "$vm"VMNIC \
   --resource-group $rgname \
   --lb-name $LBName
  done
```
#4. Update Route advertisements as necessary
</br>See [here](https://learn.microsoft.com/en-us/azure/route-server/next-hop-ip#active-active-nva-connectivity)
</br>You can deploy a set of active-active NVAs behind an internal load balancer to optimize connectivity performance. With the support for Next hop IP, you can define the next hop for both NVA instances as the IP address of the internal load balancer. Traffic that reaches the load balancer is sent to both NVA instances.

Route-maps can be used to set next-hop. Since we are sending this route-map to your remote neighbor, it should be outbound

On Hub-nva1 device, set next-hop ip to LB ip:
```
conf t
(config)#router bgp 65001
(config-router)#address-family ipv4
(config-router-af)#neighbor 10.0.3.4 route-map Change-Next-Hop out
(config-router-af)#neighbor 10.0.3.5 route-map Change-Next-Hop out

!Create the Routemap
(config)#route-map To-ARS
(config-route-map)#set ip next-hop 10.0.1.200           <-- LB private IP
end
```

On Hub-nva2 device, set next-hop ip to LB ip:
```
conf t
(config)#router bgp 65001
(config-router)#address-family ipv4
(config-router-af)#neighbor 10.0.3.4 route-map Change-Next-Hop out
(config-router-af)#neighbor 10.0.3.5 route-map Change-Next-Hop out

!Create the Routemap
(config)#route-map To-ARS
(config-route-map)#set ip next-hop 10.0.1.200           <-- LB private IP
end
```


