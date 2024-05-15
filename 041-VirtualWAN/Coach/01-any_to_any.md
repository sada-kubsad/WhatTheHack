# Challenge 1: Any-To-Any connectivity - Coach's Guide

**[Home](./README.md)** - [Next Challenge >](./02-vpn.md)

## Notes and Guidance

- You might want to pre-create VNets and VMs previous to the event to gain time

## Solution Guide

The following sections contain a possible solution based on Azure CLI

### Creating RG, VWAN and hubs

To create Virtual WAN:

```bash
# Variables
rg=wthvwan
vwan=vwan
location1=westus2
location2=eastus2
username=azureuser
password=Msft123Msft123
vwan_hub1_prefix=192.168.1.0/24
vwan_hub2_prefix=192.168.2.0/24
# Branches
publisher=cisco
offer=cisco-c8000v-byol
sku=17_13_01a-byol
version=$(az vm image list -p $publisher -f $offer -s $sku --all --query '[0].version' -o tsv)
branch1_prefix=172.16.1.0/24
branch1_subnet=172.16.1.0/26
branch1_gateway=172.16.1.1
branch1_bgp_ip=172.16.1.10
branch1_asn=65001
branch2_prefix=172.16.2.0/24
branch2_subnet=172.16.2.0/26
branch2_gateway=172.16.2.1
branch2_bgp_ip=172.16.2.10
branch2_2ary_bgp_ip=172.16.2.20
branch2_asn=65002
# Create RG, VWAN, hubs
az group create -n $rg -l $location1
# vwan and hubs
az network vwan create -n $vwan -g $rg -l $location1 --branch-to-branch-traffic true --type Standard
az network vhub create -n hub1 -g $rg --vwan $vwan -l $location1 --address-prefix $vwan_hub1_prefix
az network vhub create -n hub2 -g $rg --vwan $vwan -l $location2 --address-prefix $vwan_hub2_prefix
# Retrieve IDs of default and none RTs. We will need this when creating the connections
hub1_default_rt_id=$(az network vhub route-table show --vhub-name hub1 -g $rg -n defaultRouteTable --query id -o tsv)
hub2_default_rt_id=$(az network vhub route-table show --vhub-name hub2 -g $rg -n defaultRouteTable --query id -o tsv)
hub1_none_rt_id=$(az network vhub route-table show --vhub-name hub1 -g $rg -n noneRouteTable --query id -o tsv)
hub2_none_rt_id=$(az network vhub route-table show --vhub-name hub2 -g $rg -n noneRouteTable --query id -o tsv)
```

### Creating VNets

This code will create spokes and connect them to the Virtual Hub:

```bash
# Spoke11 in location1
spoke_id=11
vnet_prefix=10.1.1.0/24
subnet_prefix=10.1.1.0/26
az vm create -n spoke${spoke_id}-vm -g $rg -l $location1 --image Ubuntu2204 --admin-username $username --generate-ssh-keys \
    --public-ip-address spoke${spoke_id}-pip --vnet-name spoke${spoke_id}-$location1 \
    --vnet-address-prefix $vnet_prefix --subnet vm --subnet-address-prefix $subnet_prefix
az network vhub connection create -n spoke${spoke_id} -g $rg --vhub-name hub1 --remote-vnet spoke${spoke_id}-$location1 \
    --internet-security true --associated-route-table $hub1_default_rt_id --propagated-route-tables $hub1_default_rt_id

# Spoke12 in location1
spoke_id=12
vnet_prefix=10.1.2.0/24
subnet_prefix=10.1.2.0/26
az vm create -n spoke${spoke_id}-vm -g $rg -l $location1 --image Ubuntu2204 --admin-username $username --generate-ssh-keys \
    --public-ip-address spoke${spoke_id}-pip --vnet-name spoke${spoke_id}-$location1 \
    --vnet-address-prefix $vnet_prefix --subnet vm --subnet-address-prefix $subnet_prefix
az network vhub connection create -n spoke${spoke_id} -g $rg --vhub-name hub1 --remote-vnet spoke${spoke_id}-$location1 \
    --internet-security true --associated-route-table $hub1_default_rt_id --propagated-route-tables $hub1_default_rt_id

# Spoke21 in location2
spoke_id=21
vnet_prefix=10.2.1.0/24
subnet_prefix=10.2.1.0/26
az vm create -n spoke${spoke_id}-vm -g $rg -l $location2 --image Ubuntu2204 --admin-username $username --generate-ssh-keys \
    --public-ip-address spoke${spoke_id}-pip --vnet-name spoke${spoke_id}-$location2 \
    --vnet-address-prefix $vnet_prefix --subnet vm --subnet-address-prefix $subnet_prefix
az network vhub connection create -n spoke${spoke_id} -g $rg --vhub-name hub2 --remote-vnet spoke${spoke_id}-$location2 \
    --internet-security true --associated-route-table $hub2_default_rt_id --propagated-route-tables $hub2_default_rt_id

# Spoke22 in location2
spoke_id=22
vnet_prefix=10.2.2.0/24
subnet_prefix=10.2.2.0/26
az vm create -n spoke${spoke_id}-vm -g $rg -l $location2 --image Ubuntu2204 --admin-username $username --generate-ssh-keys \
    --public-ip-address spoke${spoke_id}-pip --vnet-name spoke${spoke_id}-$location2 \
    --vnet-address-prefix $vnet_prefix --subnet vm --subnet-address-prefix $subnet_prefix
az network vhub connection create -n spoke${spoke_id} -g $rg --vhub-name hub2 --remote-vnet spoke${spoke_id}-$location2 \
    --internet-security true --associated-route-table $hub2_default_rt_id --propagated-route-tables $hub2_default_rt_id

# Backdoor for access from the testing device over the Internet
myip=$(curl -s4 ifconfig.co)
az network route-table create -n spokes-$location1 -g $rg -l $location1
az network route-table route create -n mypc -g $rg --route-table-name spokes-$location1 --address-prefix "${myip}/32" --next-hop-type Internet
az network vnet subnet update -n vm --vnet-name spoke11-$location1 -g $rg --route-table spokes-$location1
az network vnet subnet update -n vm --vnet-name spoke12-$location1 -g $rg --route-table spokes-$location1
az network route-table create -n spokes-$location2 -g $rg -l $location2
az network route-table route create -n mypc -g $rg --route-table-name spokes-$location2 --address-prefix "${myip}/32" --next-hop-type Internet
az network vnet subnet update -n vm --vnet-name spoke21-$location2 -g $rg --route-table spokes-$location2
az network vnet subnet update -n vm --vnet-name spoke22-$location2 -g $rg --route-table spokes-$location2
```
## Start, Stop and Check status of VMs
```
Start All VMs:
--------------
az vm start  -g wthvwan -n spoke11-vm   > /dev/null 2>&1 & 
az vm start  -g wthvwan -n spoke12-vm   > /dev/null 2>&1 & 
az vm start  -g wthvwan -n spoke21-vm   > /dev/null 2>&1 & 
az vm start  -g wthvwan -n spoke22-vm   > /dev/null 2>&1 & 

Stop Deallocate All VMs:
-------------------------
az vm deallocate -g wthvwan -n spoke11-vm  > /dev/null 2>&1 & 
az vm deallocate -g wthvwan -n spoke12-vm  > /dev/null 2>&1 & 
az vm deallocate -g wthvwan -n spoke21-vm  > /dev/null 2>&1 & 
az vm deallocate -g wthvwan -n spoke22-vm  > /dev/null 2>&1 & 

Check the status of VMSs:
-------------------------
az vm get-instance-view -g wthvwan -n spoke11-vm    | grep -i power
az vm get-instance-view -g wthvwan -n spoke12-vm   | grep -i power
az vm get-instance-view -g wthvwan -n spoke21-vm   | grep -i power
az vm get-instance-view -g wthvwan -n spoke22-vm   | grep -i power

```
## Connecting to VMs
</BR>cat ~/.ssh/id_rsa.pub returns the public key on the local VM.
Update local VM public key to Azure VM > Reset Passsword > SSH public Key source = use Existing key 
```
My public key is:
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCY12aVXmISdwgxsWUEw7AeUdSfl9db16/dMDDDkNlhBYDeNVALZD0NncYOa8Fa/l4GHN0+2jfPuUPnQtVJwdLXUynQuMmPOQ384jhe8VzV0vd3JZW8GAxf6EZVye50ZduWl7cX/vM9OyeuAcESSwC3tuSfk8WwTijW7nvczW1z5dDuAPq6ceO3MaPe5uBy3ZaC5xhvGfPAKdbKvNXTGbNOQ/QnVsAhYAWWVVkiOILtVqxqs/p+sEHsDGDM0a/o1Qjo/M8xHKEmsbfp8VyzmXATD455H/80tnmN6KwyzYZNIP/5DDscyEhthDqxJ/p9Qd1kb42DAFO6g6dSKLuufzAJ

Username/password for access is azureuser/Msft123Maft123

export spoke11vm=$(az network public-ip show -n spoke11-pip -g wthvwan --query ipAddress -o tsv)
export spoke12vm=$(az network public-ip show -n spoke12-pip -g wthvwan --query ipAddress -o tsv)
export spoke21vm=$(az network public-ip show -n spoke21-pip -g wthvwan --query ipAddress -o tsv)
export spoke22vm=$(az network public-ip show -n spoke22-pip -g wthvwan --query ipAddress -o tsv)


Using SSH key: 
ssh azureuser@$spoke11vm
ssh azureuser@$spoke12vm
ssh azureuser@$spoke21vm
ssh azureuser@$spoke22vm

```
### VMs from different VNets on the same hub can communicate
Pings within Hub VMs should be possible
```
```
### VMs from different VNets on different hubs can communicate
By default pings of VMs across Hubs does not work. 
```
```
