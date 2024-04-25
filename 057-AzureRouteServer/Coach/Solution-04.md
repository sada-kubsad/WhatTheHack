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

#1. Deploy another instance Central NVA for High Availability
</br>Use same "Hub" Azure Virtual network but add two new subnets.
```
# Variables
echo -n "Please enter the resource group name where the VPN VNG is located: "
read RGNAME
echo -n "Please enter a password to be used for the new NVA: "
read ADMIN_PASSWORD
echo ""
rg="$RGNAME"
username=azureuser
# You may change the name and address space of the subnets if desired or required. 
nva_subnet_name=nva
nva_subnet_prefix='10.0.1.0/24'


# Try to find a VNG in the RG
echo "Searching for a VPN gateway in the resource group $rg..."
vpngw_name=$(az network vnet-gateway list -g "$rg" --query '[0].name' -o tsv)
location=$(az network vnet-gateway show -n "$vpngw_name" -g "$rg" --query location -o tsv)
vnet_name=$(az network vnet-gateway show -n "$vpngw_name" -g "$rg" --query 'ipConfigurations[0].subnet.id' -o tsv | cut -d/ -f 9)
# Create NVA
nva_name="${vnet_name}-nva1"
nva_pip_name="${vnet_name}-nva1-pip"
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
#2. Establish BGP Peering with new NVA
```
```
#3. Setup Internal Load Balancer
</br>Internal Load Balancers are required for traffic symmetry.
```
```
#4. Update Route advertisements as necessary
```
```
