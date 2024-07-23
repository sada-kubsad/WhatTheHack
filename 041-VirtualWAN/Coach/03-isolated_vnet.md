# Challenge 3: Isolated Virtual Networks - Coach's Guide

[< Previous Challenge](./02-vpn.md) - **[Home](./README.md)** - [Next Challenge >](./04-secured_hub.md)

## Notes and Guidance

- Here again, you might want to create the VNets before the actual event

## Solution Guide
## 1. Plan the Route Tables needed, their associations and propagations

### Success Criteria</B>
- The Development VNet should be able to communicate with the other Development VNet in the other hub, and to both Common Services VNets</B>
- The Production VNets should be able to communicate with the other Production VNets (same hub and across hubs), and to both Common Services VNets
- The Development VNets should not be able to communicate to the Production VNets
- All VNets should be able to communicate with the VPN branches

### Requirements translate to this:

| From | To | Dev VNets | Prod VNet | Serivces VNets | Branches |
| --- | --- | --- | --- | --- | --- |
| Dev Vnets | -> | Direct |  | Direct | Direct |
| Prod VNets | ->  |  | Direct | Direct | Direct |
| Services VNets | ->  |  |  |  | Direct |
| Branches VNets | ->  | Direct | Direct | Direct | Direct |

These translate to the need for 3 different Routing Tables, one each for Dev VNets, Prod Vnets and Services VNets.

## 2. Create VNets 3 and 4 in hubs 1 and 2

Create two additional VNets per Hub. Also connect those Vnets to their Hubs using default routing tables. We will change the routing tables (default routing tables) in a future step:

```bash
# Spoke13 in location1
spoke_id=13
vnet_prefix=10.1.3.0/24
subnet_prefix=10.1.3.0/26
az vm create -n spoke${spoke_id}-vm -g $rg -l $location1 --image Ubuntu2204 --admin-username $username --generate-ssh-keys \
    --public-ip-address spoke${spoke_id}-pip --vnet-name spoke${spoke_id}-$location1 \
    --vnet-address-prefix $vnet_prefix --subnet vm --subnet-address-prefix $subnet_prefix
az network vhub connection create -n spoke${spoke_id} -g $rg --vhub-name hub1 --remote-vnet spoke${spoke_id}-$location1 \
    --internet-security true --associated-route-table $hub1_default_rt_id --propagated-route-tables $hub1_default_rt_id
# Spoke14 in location1
spoke_id=14
vnet_prefix=10.1.4.0/24
subnet_prefix=10.1.4.0/26
az vm create -n spoke${spoke_id}-vm -g $rg -l $location1 --image Ubuntu2204 --admin-username $username --generate-ssh-keys \
    --public-ip-address spoke${spoke_id}-pip --vnet-name spoke${spoke_id}-$location1 \
    --vnet-address-prefix $vnet_prefix --subnet vm --subnet-address-prefix $subnet_prefix
az network vhub connection create -n spoke${spoke_id} -g $rg --vhub-name hub1 --remote-vnet spoke${spoke_id}-$location1 \
    --internet-security true --associated-route-table $hub1_default_rt_id --propagated-route-tables $hub1_default_rt_id
# Spoke23 in location2
spoke_id=23
vnet_prefix=10.2.3.0/24
subnet_prefix=10.2.3.0/26
az vm create -n spoke${spoke_id}-vm -g $rg -l $location2 --image Ubuntu2204 --admin-username $username --generate-ssh-keys \
    --public-ip-address spoke${spoke_id}-pip --vnet-name spoke${spoke_id}-$location2 \
    --vnet-address-prefix $vnet_prefix --subnet vm --subnet-address-prefix $subnet_prefix
az network vhub connection create -n spoke${spoke_id} -g $rg --vhub-name hub2 --remote-vnet spoke${spoke_id}-$location2 \
    --internet-security true --associated-route-table $hub2_default_rt_id --propagated-route-tables $hub2_default_rt_id
# Spoke24 in location2
spoke_id=24
vnet_prefix=10.2.4.0/24
subnet_prefix=10.2.4.0/26
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

## 3. Modify custom routing to achieve VNet isolation

### 3.1 Create new route tables:
#### 3.1.1 Create Route Tables in Hub1:
```bash
# Create separate RTs in hub1
az network vhub route-table create -n hub1DEV --vhub-name hub1 -g $rg --labels dev
az network vhub route-table create -n hub1PROD --vhub-name hub1 -g $rg --labels prod
az network vhub route-table create -n hub1CS --vhub-name hub1 -g $rg --labels cs
hub1_dev_rt_id=$(az network vhub route-table show --vhub-name hub1 -g $rg -n hub1DEV --query id -o tsv)
hub1_prod_rt_id=$(az network vhub route-table show --vhub-name hub1 -g $rg -n hub1PROD --query id -o tsv)
hub1_cs_rt_id=$(az network vhub route-table show --vhub-name hub1 -g $rg -n hub1CS --query id -o tsv)
```
#### 3.1.2 Create Route Tables in Hub2:
```
# Create separate RTs in hub2
az network vhub route-table create -n hub2DEV --vhub-name hub2 -g $rg --labels dev
az network vhub route-table create -n hub2PROD --vhub-name hub2 -g $rg --labels prod
az network vhub route-table create -n hub2CS --vhub-name hub2 -g $rg --labels cs
hub2_dev_rt_id=$(az network vhub route-table show --vhub-name hub2 -g $rg -n hub2DEV --query id -o tsv)
hub2_prod_rt_id=$(az network vhub route-table show --vhub-name hub2 -g $rg -n hub2PROD --query id -o tsv)
hub2_cs_rt_id=$(az network vhub route-table show --vhub-name hub2 -g $rg -n hub2CS --query id -o tsv)
```
### 3.2 Change association/propagation settings
```
# Setup:
# * Spoke11/21: DEV
# * Spoke12/13/22/23: PROD
# * Spoke14/24: CS
```
#### 3.2.1 Modify VNet Connections in Hub1:
In Hub1, for each Spoke, attach its associated & propagated route table.
Note: Connection <B>modifications</B> are via <B>create</B> on az network vhub route-table 
```
# Modify VNet connections in hub1:
az network vhub connection create -n spoke11 -g $rg --vhub-name hub1 --remote-vnet spoke11-$location1 --internet-security true \
    --associated-route-table $hub1_dev_rt_id --propagated-route-tables $hub1_dev_rt_id --labels dev cs default
az network vhub connection create -n spoke12 -g $rg --vhub-name hub1 --remote-vnet spoke12-$location1 --internet-security true \
    --associated-route-table $hub1_prod_rt_id --propagated-route-tables $hub1_prod_rt_id --labels prod cs default
az network vhub connection create -n spoke13 -g $rg --vhub-name hub1 --remote-vnet spoke13-$location1 --internet-security true \
    --associated-route-table $hub1_prod_rt_id --propagated-route-tables $hub1_prod_rt_id --labels prod cs default
az network vhub connection create -n spoke14 -g $rg --vhub-name hub1 --remote-vnet spoke14-$location1 --internet-security true \
    --associated-route-table $hub1_cs_rt_id --propagated-route-tables $hub1_cs_rt_id --labels dev prod cs default
```
#### 3.2.2 Modify connections in Hub2:
In Hub2, for each Spoke, attach its associated & propagated route table.
Note: Connection <B>modifications</B> are via <B>create</B> on az network vhub route-table 
```
# Modify VNet connections in hub2:
az network vhub connection create -n spoke21 -g $rg --vhub-name hub2 --remote-vnet spoke21-$location2 --internet-security true \
    --associated-route-table $hub2_dev_rt_id --propagated-route-tables $hub2_dev_rt_id --labels dev cs default
az network vhub connection create -n spoke22 -g $rg --vhub-name hub2 --remote-vnet spoke22-$location2 --internet-security true \
    --associated-route-table $hub2_prod_rt_id --propagated-route-tables $hub2_prod_rt_id --labels prod cs default
az network vhub connection create -n spoke23 -g $rg --vhub-name hub2 --remote-vnet spoke23-$location2 --internet-security true \
    --associated-route-table $hub2_prod_rt_id --propagated-route-tables $hub2_prod_rt_id --labels prod cs default
az network vhub connection create -n spoke24 -g $rg --vhub-name hub2 --remote-vnet spoke24-$location2 --internet-security true \
    --associated-route-table $hub2_cs_rt_id --propagated-route-tables $hub2_cs_rt_id --labels dev prod cs default
```
### 3.3 Modify VPN Connections
```
# Modify VPN connections
az network vpn-gateway connection create -n branch1 --gateway-name hubvpn1 -g $rg --remote-vpn-site branch1 \
    --enable-bgp true --protocol-type IKEv2 --shared-key "$password" --connection-bandwidth 100 --routing-weight 10 --internet-security true \
    --associated-route-table $hub1_default_rt_id --propagated-route-tables $hub1_default_rt_id --labels default dev prod cs
az network vpn-gateway connection create -n branch2 --gateway-name hubvpn2 -g $rg --remote-vpn-site branch2 \
    --enable-bgp true --protocol-type IKEv2 --shared-key "$password" --connection-bandwidth 100 --routing-weight 10 --internet-security true \
    --associated-route-table $hub2_default_rt_id --propagated-route-tables $hub2_default_rt_id --labels default dev prod cs
```

## 4. Connect to VMs
```
export spoke11vm=$(az network public-ip show -n spoke11-pip -g wthvwan --query ipAddress -o tsv)
export spoke12vm=$(az network public-ip show -n spoke12-pip -g wthvwan --query ipAddress -o tsv)
export spoke13vm=$(az network public-ip show -n spoke13-pip -g wthvwan --query ipAddress -o tsv)
export spoke14vm=$(az network public-ip show -n spoke14-pip -g wthvwan --query ipAddress -o tsv)

export spoke21vm=$(az network public-ip show -n spoke21-pip -g wthvwan --query ipAddress -o tsv)
export spoke22vm=$(az network public-ip show -n spoke22-pip -g wthvwan --query ipAddress -o tsv)
export spoke23vm=$(az network public-ip show -n spoke23-pip -g wthvwan --query ipAddress -o tsv)
export spoke24vm=$(az network public-ip show -n spoke24-pip -g wthvwan --query ipAddress -o tsv)

export branch1_nva_ip=$(az network public-ip show -n branch1-pip -g wthvwan --query ipAddress -o tsv)
export branch2_nva_ip=$(az network public-ip show -n branch2-pip -g wthvwan --query ipAddress -o tsv)

export branch1_nva_vm=$(az network public-ip show -n branch1-nva-VM-ip -g wthvwan --query ipAddress -o tsv)
export branch2_nva_vm=$(az network public-ip show -n branch2-nva-VM-ip -g wthvwan --query ipAddress -o tsv)

ssh azureuser@$spoke11vm
ssh azureuser@$branch2_nva_ip

My public key is:
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCY12aVXmISdwgxsWUEw7AeUdSfl9db16/dMDDDkNlhBYDeNVALZD0NncYOa8Fa/l4GHN0+2jfPuUPnQtVJwdLXUynQuMmPOQ384jhe8VzV0vd3JZW8GAxf6EZVye50ZduWl7cX/vM9OyeuAcESSwC3tuSfk8WwTijW7nvczW1z5dDuAPq6ceO3MaPe5uBy3ZaC5xhvGfPAKdbKvNXTGbNOQ/QnVsAhYAWWVVkiOILtVqxqs/p+sEHsDGDM0a/o1Qjo/M8xHKEmsbfp8VyzmXATD455H/80tnmN6KwyzYZNIP/5DDscyEhthDqxJ/p9Qd1kb42DAFO6g6dSKLuufzAJ

```

