# Challenge 03 - Connect Network Virtual Appliance to an SDWAN environment - Coach's Guide 

[< Previous Solution](./Solution-02.md) - **[Home](./README.md)** - [Next Solution >](./Solution-04.md)

## Notes & Guidance
- Please remind the students that SDWAN in this scenario will be represented with TWO different Simulated On Premises environments. SDWAN is a technology that most students will require licenses from a specific vendor and specific knowledge, which is not required for these testing purposes.
- The SDWAN environments will be simulated utilizing two separate VNets with Two different Cisco NVAs on each.
- Each Vnet needs to be created on a different Region and Resource Group than the main Hub and Spoke. These are not arbitrary; we want the closest Region to the Hub and Spoke to have preference over the other.
- The configuration templates are given to the student.
- Please make sure no Overlapping Address space is occupied on those two VNets and utilized on the Virtual Tunnel Interfaces within the Cisco NVAs. For example, do not utilize a Cisco VTI with 10.0.0.1 since Azure, is the first usable of a given subnet.
- Establish One IPSec tunnel from each of these two SDWAN simulated Cisco Virtual Appliances, to the Cisco CSR Central Virtual Appliance in the Hub Virtual Network.
- Establish BGP from each of these two SDWAN simulated Cisco Virtual Appliances to Cisco CSR Central Virtual Appliance in the Hub Virtual Network.
- Guide the Student through advertising identical address spaces from the two SDWAN Virtual Appliances via BGP. Students will need some guidance to understand how to configure the loopbacks to advertise such IPs, and how to configure route maps and ip access list to have better preference using BGP attributes. For example, you can have the student advertise 1.1.1.1/32 created as a loopback interface.
- First you might announce the same prefixes with Equal attributes and see how it reflects across the effective routes of the Hub and Spoke, on Premises. Also, please look at the Received and Advertised prefixes from the Route Server's perspective. Lastly, you may also check the routing table on the Virtual Appliance themselves.
- The main idea is that the closest Region to the Hub and Spoke is more desirable than the Region further away from the Hub and Spoke. In other words, manipulate the BGP attributes within the SDWAN NVAs to ensure the latter.
- Finally, look at the effective route tables from all the VMs on the Hub and Spoke Topology and help the student understand how Route Server installs the preferred route.

# My Solution
## 1. Create 2 Vnets in in different Region and Resource Group than the main Hub and Spoke 
In a later excercise, we will want the closest Region to the Hub and Spoke to have preference over the other. Since main Hub and Spoke is in West-US3, create 2 Vnet in different regions one each in West-US2 and East-US2 since 
Make sure no Overlapping Address space is occupied on those two VNETs

Commands below have been customized from script provided in [sdwancsr.md in the resources folder](../Student/Resources/sdwancsr.md):

## 1.1 SD-WAN Onprem Site 1:

```bash
# Variables
rg1=wthars-SDWAN-OnPrem-1
location1=westus2
vnet_name1=SDWAN-OnPrem-1

Vnet_address_prefix1=10.11.0.0/16
Vnet_out_subnet_name1=sdwan1outsidesubnet
vnet_out_subnet1=10.11.1.0/24
Vnet_in_subnet_name1=sdwan1insidesidesubnet
vnet_in_subnet1=10.11.2.0/24

# Create the resource group, VNET and subnet:
az group create --name $rg1 --location $location1
az network vnet create --name $vnet_name1 --resource-group $rg1 --address-prefix $Vnet_address_prefix1
az network vnet subnet create --address-prefix $vnet_out_subnet1 --name $Vnet_out_subnet_name1 --resource-group $rg1 --vnet-name $vnet_name1
az network vnet subnet create --address-prefix $vnet_in_subnet1 --name $Vnet_in_subnet_name1 --resource-group $rg1 --vnet-name $vnet_name1

```

## 1.2 OnPrem Site 2:
```bash

# Variables
rg2=wthars-SDWAN-OnPrem-2
location2=eastus2
vnet_name2=SDWAN-OnPrem-2

Vnet_address_prefix2=10.12.0.0/16
Vnet_out_subnet_name2=SDWAN2outsidesubnet
vnet_out_subnet2=10.12.1.0/24
Vnet_in_subnet_name2=SDWAN2insidesubnet
vnet_in_subnet2=10.12.2.0/24

# Create the resource group, VNET and subnet:
az group create --name $rg2 --location $location2
az network vnet create --name $vnet_name2 --resource-group $rg2 --address-prefix $Vnet_address_prefix2
az network vnet subnet create --address-prefix $vnet_out_subnet2 --name $Vnet_out_subnet_name2 --resource-group $rg2 --vnet-name $vnet_name2
az network vnet subnet create --address-prefix $vnet_in_subnet2 --name $Vnet_in_subnet_name2 --resource-group $rg2 --vnet-name $vnet_name2



```

## 2. Create two new NVAs simulating on-prem
Use the provided configuration templates
Make sure no Overlapping Address space is occupied on the Virtual Tunel INterfaces within the Cisco NVAs. Eg: do not utilize a Cisco VTI with 10.0.0.1 since Azure, is the first usable of a given subnet.

## 2.1 Onprem Site 1:
```bash
#Create NSG for SDWAN1 Cisco CSR 8000V
az network nsg create --resource-group $rg1 --name SDWAN1-NSG --location $location1
az network nsg rule create --resource-group $rg1 --nsg-name SDWAN1-NSG --name all --access Allow --protocol "*" --direction Inbound --priority 100 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*"


# Create SDWAN Router Site 1:
az network public-ip create --name SDWAN1PublicIP --resource-group $rg1 --idle-timeout 30 --allocation-method Static
az network nic create --name SDWAN1OutsideInterface --resource-group $rg1 --subnet $Vnet_out_subnet_name1 --vnet $vnet_name1 --public-ip-address SDWAN1PublicIP --ip-forwarding true --network-security-group SDWAN1-NSG
az network nic create --name SDWAN1InsideInterface --resource-group $rg1 --subnet $Vnet_in_subnet_name1 --vnet $vnet_name1 --ip-forwarding true --network-security-group SDWAN1-NSG
az vm image terms accept --urn cisco:cisco-c8000v-byol:17_13_01a-byol:latest
az vm create --resource-group $rg1 --location $location1 --name SDWAN1Router --size Standard_D2_v2 --nics SDWAN1OutsideInterface SDWAN1InsideInterface  --image cisco:cisco-c8000v-byol:17_13_01a-byol:latest --admin-username azureuser --admin-password Msft123Msft123 --no-wait

```

## 2.2 OnPrem Site 2:
```bash
# Create NSG for SDWAN2 Cisco CSR 8000V
az network nsg create --resource-group $rg2 --name SDWAN2-NSG --location $location2
az network nsg rule create --resource-group $rg2 --nsg-name SDWAN2-NSG --name all --access Allow --protocol "*" --direction Inbound --priority 100 --source-address-prefix "*" --source-port-range "*" --destination-address-prefix "*" --destination-port-range "*"


#Create SDWAN Router Site 2
az network public-ip create --name SDWAN2PublicIP --resource-group $rg2 --idle-timeout 30 --allocation-method Static
az network nic create --name SDWAN2OutsideInterface --resource-group $rg2 --subnet $Vnet_out_subnet_name2 --vnet $vnet_name2 --public-ip-address SDWAN2PublicIP --ip-forwarding true --network-security-group SDWAN2-NSG
az network nic create --name SDWAN2InsideInterface --resource-group $rg2 --subnet $Vnet_in_subnet_name2 --vnet $vnet_name2 --ip-forwarding true --network-security-group SDWAN2-NSG
az vm image accept-terms --urn cisco:cisco-c8000v-byol:17_13_01a-byol:latest
az vm create --resource-group $rg2 --location $location2 --name SDWAN2Router --size Standard_D2_v2 --nics SDWAN2OutsideInterface SDWAN2InsideInterface  --image cisco:cisco-c8000v-byol:17_13_01a-byol:latest --admin-username azureuser --admin-password Msft123Msft123 --no-wait

```

## 3. Manage the SDWAN NVAs
### 3.1 Enable required licenses on the SDWAN
On Each SDWAN appliances,run:
```bash
config t
license boot level network-essentials addon dna-essentials
```
This is required so that the crypto ikev2... command gets enabled.  
**Then reboot the SDWANs**

### 3.2 Start the SDWAN NVAs
```bash
az vm start  -g wthars-SDWAN-OnPrem-1 -n SDWAN1Router        > /dev/null 2>&1 & 
az vm start  -g wthars-SDWAN-OnPrem-2 -n SDWAN2Router        > /dev/null 2>&1 &

az vm start  -g wthars -n hub-nva1        > /dev/null 2>&1 &
az vm start  -g wthars -n datacenter-nva  > /dev/null 2>&1 & 

```

### 3.3 Stop Deallocate the SDWAN NVAs
```bash
az vm deallocate -g wthars-SDWAN-OnPrem-1 -n SDWAN1Router  > /dev/null 2>&1 &
az vm deallocate -g wthars-SDWAN-OnPrem-2 -n SDWAN2Router  > /dev/null 2>&1 &

az vm deallocate -g wthars -n hub-nva1         > /dev/null 2>&1 &
az vm deallocate -g wthars -n datacenter-nva   > /dev/null 2>&1 & 
```

### 3.4 Check State the SDWAN NVAs
```bash
az vm get-instance-view -g wthars-SDWAN-OnPrem-1 -n  SDWAN1Router  | grep -i power
az vm get-instance-view -g wthars-SDWAN-OnPrem-2 -n  SDWAN2Router  | grep -i power

az vm get-instance-view -g wthars -n  hub-nva1      | grep -i power
az vm get-instance-view -g wthars -n datacenter-nva | grep -i power
```
### 3.5 Connect to the SDWAN NVAs:

```bash
export onpremSDWAN_1=$(az network public-ip show -n SDWAN1PublicIP -g wthars-SDWAN-OnPrem-1 --query ipAddress -o tsv)
ssh azureuser@$onpremSDWAN_1 -oHostKeyAlgorithms=+ssh-rsa -oKexAlgorithms=+diffie-hellman-group14-sha1

export onpremSDWAN_2=$(az network public-ip show -n SDWAN2PublicIP -g wthars-SDWAN-OnPrem-2 --query ipAddress -o tsv)
ssh azureuser@$onpremSDWAN_2 -oHostKeyAlgorithms=+ssh-rsa -oKexAlgorithms=+diffie-hellman-group14-sha1

export hubnva=$(az network public-ip show -n hub-nva1-pip -g wthars --query ipAddress -o tsv)
ssh azureuser@$hubnva -oHostKeyAlgorithms=+ssh-rsa -oKexAlgorithms=+diffie-hellman-group14-sha1

export datacenter=$(az network public-ip show -n datacenter-pip -g wthars --query ipAddress -o tsv)
ssh azureuser@$datacenter -oHostKeyAlgorithms=+ssh-rsa -oKexAlgorithms=+diffie-hellman-group14-sha1

```


## 4. Establish One IPSec tunnel from each of these two SDWAN simulated Cisco Virtual Appliances, to the Cisco CSR Central Virtual Appliance in the Hub Virtual Network.
Establish BGP from each of these two SDWAN simulated Cisco Virtual Appliances to Cisco CSR Central Virtual Appliance in the Hub Virtual Network.

See [diagram here](../MyAssets/SDWAN-VPN-Connectivity.pdf) for values that get plugged into the scrip below

### 4.0 Central NVA to SDWAN Routers Cisco CSR 8000v BGP over IPsec Connection
```
crypto ikev2 proposal to-sdwan-proposal
  encryption aes-cbc-128 aes-cbc-256
  integrity sha1 sha256 sha384 sha512 
  group 14 15 16 
  exit

crypto ikev2 policy to-sdwan-policy
  proposal to-sdwan-proposal
  match address local 10.0.1.4
  exit
  
crypto ikev2 keyring to-sdwan-keyring
  peer 52.149.9.59
    address 52.149.9.59
    pre-shared-key Msft123Msft123
    exit
  peer 20.186.73.55
    address 20.186.73.55
    pre-shared-key Msft123Msft123
    exit
  exit

crypto ikev2 profile to-sdwan-profile
  match address local 10.0.1.4
  match identity remote address 10.11.1.4 255.255.255.255
  match identity remote address 10.12.1.4 255.255.255.255
  authentication remote pre-share
  authentication local  pre-share
  lifetime 3600
  dpd 10 5 on-demand
  keyring local to-sdwan-keyring
  exit

crypto ipsec transform-set to-sdwan-TransformSet esp-gcm 256 
  mode tunnel
  exit

crypto ipsec profile to-sdwan-IPsecProfile
  set transform-set to-sdwan-TransformSet
  set ikev2-profile to-sdwan-profile
  set security-association lifetime seconds 3600
  exit

interface tunnel 98
  description to SDWAN1-Router
  ip address 192.168.1.1 255.255.255.255
  tunnel mode ipsec ipv4
  ip tcp adjust-mss 1350
  tunnel source GigabitEthernet1
  tunnel destination 52.149.9.59
  tunnel protection ipsec profile to-sdwan-IPsecProfile
  exit 
  
 interface tunnel 99 
  description to SDWAN2-Router
  ip address 192.168.1.4 255.255.255.255
  tunnel mode ipsec ipv4
  ip tcp adjust-mss 1350
  tunnel source GigabitEthernet1
  tunnel destination 20.186.73.55
  tunnel protection ipsec profile to-sdwan-IPsecProfile
  exit  

router bgp 65001
  bgp log-neighbor-changes
  neighbor 192.168.1.2 remote-as 65002
  neighbor 192.168.1.2 ebgp-multihop 255
  neighbor 192.168.1.2 update-source tunnel 98
  !
  neighbor 192.168.1.3 remote-as 65003
  neighbor 192.168.1.3 ebgp-multihop 255
  neighbor 192.168.1.3 update-source tunnel 99
  

  address-family ipv4
   neighbor 192.168.1.2 activate 
   neighbor 192.168.1.3 activate 
    exit
  exit

!route BGP peer IP over the tunnel
ip route 192.168.1.2 255.255.255.255 Tunnel 98
ip route 192.168.1.3 255.255.255.255 Tunnel 99
```

### 4.1 SDWAN1 Router to Central NVA Cisco CSR 8000v BGP over IPsec Connection
```
crypto ikev2 proposal to-central-nva-proposal
  encryption aes-cbc-128 aes-cbc-256
  integrity sha1 sha256 sha384 sha512 
  group 14 15 16 
  exit

crypto ikev2 policy to-central-nva-policy
  proposal to-central-nva-proposal
  match address local 10.11.1.4
  exit
  
crypto ikev2 keyring to-central-nva-keyring
  peer 20.163.50.108
    address 20.163.50.108
    pre-shared-key Msft123Msft123
    exit
  exit

crypto ikev2 profile to-central-nva-profile
  match address local 10.11.1.4
  match identity remote address 10.0.1.4 255.255.255.255
  authentication remote pre-share
  authentication local  pre-share
  lifetime 3600
  dpd 10 5 on-demand
  keyring local to-central-nva-keyring
  exit

crypto ipsec transform-set to-central-nva-TransformSet esp-gcm 256 
  mode tunnel
  exit

crypto ipsec profile to-central-nva-IPsecProfile
  set transform-set to-central-nva-TransformSet
  set ikev2-profile to-central-nva-profile
  set security-association lifetime seconds 3600
  exit

interface tunnel 98
  ip address 192.168.1.2 255.255.255.255
  tunnel mode ipsec ipv4
  ip tcp adjust-mss 1350
  tunnel source GigabitEthernet1
  tunnel destination 20.163.50.108
  tunnel protection ipsec profile to-central-nva-IPsecProfile
  exit

router bgp 65002
  bgp log-neighbor-changes
  neighbor 192.168.1.1 remote-as 65001
  neighbor 192.168.1.1 ebgp-multihop 255
  neighbor 192.168.1.1 update-source tunnel 98
  
  address-family ipv4
    network 10.11.0.0 mask 255.255.0.0
    redistribute connected
    neighbor 192.168.1.1 activate    
    exit
  exit

!route BGP peer IP over the tunnel
ip route 192.168.1.1 255.255.255.255 Tunnel 98
ip route 10.11.0.0 255.255.0.0 Null0
```

### 4.2 SDWAN2 Router to Central NVA Cisco CSR 8000 BGP over IPsec Connection
```
crypto ikev2 proposal to-central-nva-proposal
  encryption aes-cbc-128 aes-cbc-256
  integrity sha1 sha256 sha384 sha512 
  group 14 15 16 
  exit

crypto ikev2 policy to-central-nva-policy
  proposal to-central-nva-proposal
  match address local 10.12.1.4
  exit
  
crypto ikev2 keyring to-central-nva-keyring
  peer 20.163.50.108
    address 20.163.50.108
    pre-shared-key Msft123Msft123
    exit
  exit

crypto ikev2 profile to-central-nva-profile
  match address local 10.12.1.4
  match identity remote address 10.0.1.4 255.255.255.255
  authentication remote pre-share
  authentication local  pre-share
  lifetime 3600
  dpd 10 5 on-demand
  keyring local to-central-nva-keyring
  exit

crypto ipsec transform-set to-central-nva-TransformSet esp-gcm 256 
  mode tunnel
  exit

crypto ipsec profile to-central-nva-IPsecProfile
  set transform-set to-central-nva-TransformSet
  set ikev2-profile to-central-nva-profile
  set security-association lifetime seconds 3600
  exit

interface tunnel 99
  ip address 192.168.1.3 255.255.255.255
  tunnel mode ipsec ipv4
  ip tcp adjust-mss 1350
  tunnel source GigabitEthernet1
  tunnel destination 20.163.50.108
  tunnel protection ipsec profile to-central-nva-IPsecProfile
  exit

router bgp 65003
  bgp log-neighbor-changes
  neighbor 192.168.1.4 remote-as 65001
  neighbor 192.168.1.4 ebgp-multihop 255
  neighbor 192.168.1.4 update-source tunnel 99

  address-family ipv4
    network 10.12.0.0 mask 255.255.0.0
    redistribute connected
    neighbor 192.168.1.1 activate    
    exit
  exit

!route BGP peer IP over the tunnel
ip route 192.168.1.4 255.255.255.255 Tunnel 99
ip route 10.12.0.0 255.255.0.0 Null0
```

## 5. Advertise identical address spaces from the two SDWAN Virtual Appliances via BGP
### 5.1 Configure the loopbacks to advertise such IPs
For example, let create 1.1.1.1/32 as a loopback address on both SDWAN 1 and SDWAN 2
Then advertise 1.1.1.1/32 created as the loopback interface.

On SDWAN 1:
```
conf t
!
interface Loopback0
  ip address 1.1.1.1 255.255.255.255
  no shutdown
!
router bgp 65002
 network 1.1.1.1 mask 255.255.255.255
end
```

On SDWAN 2:
```
conf t
!
interface Loopback0
  ip address 1.1.1.1 255.255.255.255
  no shutdown
!
router bgp 65003
 network 1.1.1.1 mask 255.255.255.255
end

```
### 5.2 Check what Routing preferences are set
#### On Central NVA:
Routes received from SDWAN 1:
```
show ip bgp neighbors 192.168.1.2 advertised-routes

     Network          Next Hop            Metric LocPrf Weight Path
 *>   1.1.1.1/32       192.168.1.2              0             0 65002 i
 *>   10.11.0.0/16     192.168.1.2              0             0 65002 i
 *>   10.11.1.0/24     192.168.1.2              0             0 65002 ?
 *>   10.12.0.0/16     192.168.1.3              0             0 65003 i
 *>   10.12.1.0/24     192.168.1.3              0             0 65003 ?
 r>   192.168.1.2/32   192.168.1.2              0             0 65002 ?
 r>   192.168.1.3/32   192.168.1.3              0             0 65003 ?

Total number of prefixes 7
```
Routes received from SDWAM 2:
```
show ip bgp neighbors 192.168.1.3 advertised-routes

     Network          Next Hop            Metric LocPrf Weight Path
 *>   1.1.1.1/32       192.168.1.2              0             0 65002 i
 *>   10.11.0.0/16     192.168.1.2              0             0 65002 i
 *>   10.11.1.0/24     192.168.1.2              0             0 65002 ?
 *>   10.12.0.0/16     192.168.1.3              0             0 65003 i
 *>   10.12.1.0/24     192.168.1.3              0             0 65003 ?
 r>   192.168.1.2/32   192.168.1.2              0             0 65002 ?
 r>   192.168.1.3/32   192.168.1.3              0             0 65003 ?

Total number of prefixes 7
```

#### On SDWAN 1:
```
show ip bgp neighbors 192.168.1.1 advertised-routes

     Network          Next Hop            Metric LocPrf Weight Path
 *>   1.1.1.1/32       0.0.0.0                  0         32768 i
 *>   10.11.0.0/16     0.0.0.0                  0         32768 i
 *>   10.11.1.0/24     0.0.0.0                  0         32768 ?
 *>   192.168.1.2/32   0.0.0.0                  0         32768 ?

Total number of prefixes 4
```
```
show ip bgp neighbors 192.168.1.4 advertised-routes
% No such neighbor or address family
```

#### On SDWAN 2:
```
show ip bgp neighbors 192.168.1.4 advertised-routes

     Network          Next Hop            Metric LocPrf Weight Path
 *>   1.1.1.1/32       0.0.0.0                  0         32768 i
 *>   10.12.0.0/16     0.0.0.0                  0         32768 i
 *>   10.12.1.0/24     0.0.0.0                  0         32768 ?
 *>   192.168.1.3/32   0.0.0.0                  0         32768 ?

Total number of prefixes 4
```

```
show ip bgp neighbors 192.168.1.1 advertised-routes
% No such neighbor or address family
```

### On ARS:

#### ARS learned from NVA:
```
az network routeserver peering list-learned-routes --routeserver ARSHack -n HubCentralNVA --query 'RouteServiceRole_IN_0' -o table
```

#### ARS advertised to NVA:
```
az network routeserver peering list-advertised-routes --routeserver ARSHack -n HubCentralNVA --query 'RouteServiceRole_IN_0' -o table
```

### On OnPrem CSR:
```
show ip bgp neighbors 10.0.0.4 advertised-routes

```

```
show ip bgp neighbors 10.0.0.5 advertised-routes
```

### 5.3 Configure route maps and ip access list to have better preference using BGP attributes. 
#### 5.3.1 Configure Route maps for better preference using BGP attributes
```

```
#### 5.3.2 Configure ip access list for better preference using BGP attributes
```

```
## 6. Announce the same prefixes with Equal attributes and see how it reflects across the effective routes of the Hub and Spoke, on Premises. 

### 6.1 Look at the Received and Advertised prefixes from the Route Server's perspective. 

### 6.2 Check the routing table on the Virtual Appliance themselves.

## 7. manipulate the BGP attributes within the SDWAN NVAs to ensure the closest Region to the Hub and Spoke  more desirable than the Region further away from the Hub and Spoke.

## 8. Look at the effective route tables from all the VMs on the Hub and Spoke Topology 
The goal is to understand how Route Server installs the preferred route
