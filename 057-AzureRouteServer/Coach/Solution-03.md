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

```bash
az group create \
    --name wthars-OnPrem-1 \
    --location westus2
az network vnet create \
    --name OnPrem-1 \
    --resource-group wthars-OnPrem-1 \
    --address-prefix 10.1.0.0/16 \
    --subnet-name OnPrem-1 \
    --subnet-prefixes 10.1.0.0/24

az group create \
    --name wthars-OnPrem-2 \
    --location eastus2
az network vnet create \
    --name OnPrem-2 \
    --resource-group wthars-OnPrem-2 \
    --address-prefix 10.2.0.0/16 \
    --subnet-name OnPrem-2 \
    --subnet-prefixes 10.2.0.0/24



## 2. Configure the two newly create NVAs
Use the provided configuration templates
Make sure no Overlapping Address space is occupied on the Virtual Tunel INterfaces within teh Cisco NVAs. Eg: do not utilize a Cisco VTI with 10.0.0.1 since Azure, is the first usable of a given subnet.

## 3. Establish One IPSec tunnel from each of these two SDWAN simulated Cisco Virtual Appliances, to the Cisco CSR Central Virtual Appliance in the Hub Virtual Network.

## 4. Establish BGP from each of these two SDWAN simulated Cisco Virtual Appliances to Cisco CSR Central Virtual Appliance in the Hub Virtual Network.

## 5. Advertise identical address spaces from the two SDWAN Virtual Appliances via BGP
### 5.1 Configure the loopbacks to advertise such IPs, 
### 5.2 Configure route maps and ip access list to have better preference using BGP attributes. 
For example, you can  advertise 1.1.1.1/32 created as a loopback interface.

## 6. Announce the same prefixes with Equal attributes and see how it reflects across the effective routes of the Hub and Spoke, on Premises. 

## 6.1 Look at the Received and Advertised prefixes from the Route Server's perspective. 

## 6.2 Check the routing table on the Virtual Appliance themselves.

## 7. manipulate the BGP attributes within the SDWAN NVAs to ensure the closest Region to the Hub and Spoke  more desirable than the Region further away from the Hub and Spoke.

## 8. Look at the effective route tables from all the VMs on the Hub and Spoke Topology 
The goal is to understand how Route Server installs the preferred route
