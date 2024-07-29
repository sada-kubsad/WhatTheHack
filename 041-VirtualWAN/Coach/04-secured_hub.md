# Challenge 4: Virtual Secure Hub - Coach's Guide

[< Previous Challenge](./03-isolated_vnet.md) - **[Home](./README.md)** - [Next Challenge >](./05-nva.md)

## Notes and Guidance
Previously: 
- No multi-region support at the time of this writing, inter-region traffic is going to be silently dropped
- Branch-to-branch traffic is not secured by the Azure Firewall at this time

## Solution Guide

### 1. Create Azure Firewalls Policy

The policy has some sample rules to allow test traffic, you might need to change them if using different protocols to test:

```bash
# Create Azure Firewall policy with sample policies
azfw_policy_name=vwanfwpolicy
az network firewall policy create -n $azfw_policy_name -g $rg
az network firewall policy rule-collection-group create -n ruleset01 --policy-name $azfw_policy_name -g $rg --priority 100
# Allow SSH
echo "Creating rule to allow SSH..."
az network firewall policy rule-collection-group collection add-filter-collection --policy-name $azfw_policy_name --rule-collection-group-name ruleset01 -g $rg \
    --name mgmt --collection-priority 101 --action Allow --rule-name allowSSH --rule-type NetworkRule --description "TCP 22" \
    --destination-addresses 10.0.0.0/8 1.1.1.1/32 2.2.2.2/32 3.3.3.3/32 --source-addresses 10.0.0.0/8 1.1.1.1/32 2.2.2.2/32 3.3.3.3/32 --ip-protocols TCP --destination-ports 22
# Allow ICMP
# echo "Creating rule to allow ICMP..."
# az network firewall policy rule-collection-group collection add-filter-collection --policy-name $azfw_policy_name --rule-collection-group-name ruleset01 -g $rg \
#     --name icmp --collection-priority 102 --action Allow --rule-name allowICMP --rule-type NetworkRule --description "ICMP traffic" \
#     --destination-addresses 10.0.0.0/8 1.1.1.1/32 2.2.2.2/32 3.3.3.3/32 --source-addresses 10.0.0.0/8 1.1.1.1/32 2.2.2.2/32 3.3.3.3/32 --ip-protocols ICMP --destination-ports "1-65535" >/dev/null
# Allow NTP
echo "Creating rule to allow NTP..."
az network firewall policy rule-collection-group collection add-filter-collection --policy-name $azfw_policy_name --rule-collection-group-name ruleset01 -g $rg \
    --name ntp --collection-priority 103 --action Allow --rule-name allowNTP --rule-type NetworkRule --description "Egress NTP traffic" \
    --destination-addresses '*' --source-addresses "10.0.0.0/8" --ip-protocols UDP --destination-ports "123"
# Example application collection with 2 rules (ipconfig.co, api.ipify.org)
echo "Creating rule to allow ifconfig.co and api.ipify.org..."
az network firewall policy rule-collection-group collection add-filter-collection --policy-name $azfw_policy_name --rule-collection-group-name ruleset01 -g $rg \
    --name ifconfig --collection-priority 201 --action Allow --rule-name allowIfconfig --rule-type ApplicationRule --description "ifconfig" \
    --target-fqdns "ifconfig.co" --source-addresses "10.0.0.0/8" --protocols Http=80 Https=443
az network firewall policy rule-collection-group collection rule add -g $rg --policy-name $azfw_policy_name --rule-collection-group-name ruleset01 --collection-name ifconfig \
    --name ipify --target-fqdns "api.ipify.org" --source-addresses "10.0.0.0/8" --protocols Http=80 Https=443 --rule-type ApplicationRule
# Example application collection with wildcards (*.ubuntu.com)
echo "Creating rule to allow *.ubuntu.com..."
az network firewall policy rule-collection-group collection add-filter-collection --policy-name $azfw_policy_name --rule-collection-group-name ruleset01 -g $rg \
    --name ubuntu --collection-priority 202 --action Allow --rule-name repos --rule-type ApplicationRule --description "ubuntucom" \
    --target-fqdns '*.ubuntu.com' --source-addresses "10.0.0.0/8" --protocols Http=80 Https=443
```

### 2. Create Azure Firewalls

The next step is creating the firewalls and attach them to the firewall policy.
These commands will take time to execute.

```bash
# Create Azure Firewalls in the virtual hubs
az network firewall create -n azfw1 -g $rg --vhub hub1 --policy $azfw_policy_name -l $location1 --sku AZFW_Hub --public-ip-count 1
az network firewall create -n azfw2 -g $rg --vhub hub2 --policy $azfw_policy_name -l $location2 --sku AZFW_Hub --public-ip-count 1
### consider Adding --tier Basic to the above commands?

# Configure static routes to firewall
azfw1_id=$(az network firewall show -n azfw1 -g $rg --query id -o tsv)
azfw2_id=$(az network firewall show -n azfw2 -g $rg --query id -o tsv)
az network vhub route-table route add -n defaultRouteTable --vhub-name hub1 -g $rg \
    --route-name default --destination-type CIDR --destinations "0.0.0.0/0" "10.0.0.0/8" "172.16.0.0/12" \
    --next-hop-type ResourceId --next-hop $azfw1_id
az network vhub route-table route add -n defaultRouteTable --vhub-name hub2 -g $rg \
    --route-name default --destination-type CIDR --destinations "0.0.0.0/0" "10.0.0.0/8" "172.16.0.0/12" \
    --next-hop-type ResourceId --next-hop $azfw2_id
```

### 3. Configure logging (optional)

Logging is critical for troubleshooting packet drops in the firewalls:

```bash
logws_name=$(az monitor log-analytics workspace list -g $rg --query '[0].name' -o tsv)
if [[ -z "$logws_name" ]]
then
    logws_name=vwanlogs$RANDOM
    echo "Creating log analytics workspace $logws_name..."
    az monitor log-analytics workspace create -n $logws_name -g $rg -l $location1
fi
logws_id=$(az resource list -g $rg -n $logws_name --query '[].id' -o tsv)
logws_customerid=$(az monitor log-analytics workspace show -n $logws_name -g $rg --query customerId -o tsv)
# VPN gateways
echo "Configuring VPN gateways..."
gw_id_list=$(az network vpn-gateway list -g $rg --query '[].id' -o tsv)
while IFS= read -r gw_id; do
    az monitor diagnostic-settings create -n mydiag --resource $gw_id --workspace $logws_id \
        --metrics '[{"category": "AllMetrics", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false } }]' \
        --logs '[{"category": "GatewayDiagnosticLog", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false}}, 
                {"category": "TunnelDiagnosticLog", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false}},
                {"category": "RouteDiagnosticLog", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false}},
                {"category": "IKEDiagnosticLog", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false}}]' >/dev/null
done <<< "$gw_id_list"
# Azure Firewalls
echo "Configuring Azure Firewalls..."
fw_id_list=$(az network firewall list -g $rg --query '[].id' -o tsv)
while IFS= read -r fw_id; do
    az monitor diagnostic-settings create -n mydiag --resource $fw_id --workspace $logws_id --export-to-resource-specific\
        --metrics '[{"category": "AllMetrics", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false } }]' \
        --logs '[{"category": "AzureFirewallApplicationRule", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false}}, 
                {"category": "AzureFirewallNetworkRule", "enabled": true, "retentionPolicy": {"days": 0, "enabled": false}}]' >/dev/null
done <<< "$fw_id_list"
```
## Success Criteria
Prove that certain traffic (e.g. ping, SSH or HTTP) is either blocked by the firewall or allowed through. Confirm this for each of the configured flows (VHV, VHB and V2I)

### 1. Check if ping is blocked or allowed through FW
#### 1.1 From Hub1:
```
#check ping from Hub1-Dev to Hub1-Prod/Services/Branches/Internet
Example: ping 10.1.1.4. 10.1.2.4, 10.1.3.4, 10.1.4.4. 172.16.1.10, 172.16.2.10
ping 10.1.1.4. 10.1.2.4, 10.1.3.4, 10.1.4.4. 172.16.1.10, 172.16.2.10
#Check ping from Hub1-Prod to Hub1-Dev/Services/Branches/Internet
#Check ping from Hub1-Services to Hub1-Dev/Prod/Branches/Internet
#Check ping from Hub1-Branches to Hub1-Dev/Prod/Services/Internet

#check ping from Hub1-Dev to Hub2-Prod/Services/Branches/Internet
#Check ping from Hub1-Prod to Hub2-Dev/Services/Branches/Internet
#Check ping from Hub1-Services to Hub2-Dev/Prod/Branches/Internet
#Check ping from Hub1-Branches to Hub2-Dev/Prod/Services/Internet
```
#### 1.2 From Hub2:
```
#check ping from Hub2-Dev to Hub1-Prod/Services/Branches/Internet
#Check ping from Hub2-Prod to Hub1-Dev/Services/Branches/Internet
#Check ping from Hub2-Services to Hub1-Dev/Prod/Branches/Internet
#Check ping from Hub2-Branches to Hub1-Dev/Prod/Services/Internet

#check ping from Hub2-Dev to Hub2-Prod/Services/Branches/Internet
#Check ping from Hub2-Prod to Hub2-Dev/Services/Branches/Internet
#Check ping from Hub2-Services to Hub2-Dev/Prod/Branches/Internet
#Check ping from Hub2-Branches to Hub2-Dev/Prod/Services/Internet
```


### 2. Check if SSH is blocked or allowed through FW
#### 2.1 From Hub1:
```
#check SSH from Hub1-Dev to Hub1-Prod/Services/Branches/Internet
ssh azureuser@10.1.1.4, azureuser@10.1.2.4, azureuser@10.1.3.4, azureuser@10.1.4.4, azureuser@10.1.1.4 
#Check SSH from Hub1-Prod to Hub1-Dev/Services/Branches/Internet
#Check SSH from Hub1-Services to Hub1-Dev/Prod/Branches/Internet
#Check SSH from Hub1-Branches to Hub1-Dev/Prod/Services/Internet

#check SSH from Hub1-Dev to Hub2-Prod/Services/Branches/Internet
#Check SSH from Hub1-Prod to Hub2-Dev/Services/Branches/Internet
#Check SSH from Hub1-Services to Hub2-Dev/Prod/Branches/Internet
#Check SSH from Hub1-Branches to Hub2-Dev/Prod/Services/Internet
```
#### 2.2 From Hub2:
```
#check SSH from Hub2-Dev to Hub1-Prod/Services/Branches/Internet
#Check SSH from Hub2-Prod to Hub1-Dev/Services/Branches/Internet
#Check SSH from Hub2-Services to Hub1-Dev/Prod/Branches/Internet
#Check SSH from Hub2-Branches to Hub1-Dev/Prod/Services/Internet

#check SSH from Hub2-Dev to Hub2-Prod/Services/Branches/Internet
#Check SSH from Hub2-Prod to Hub2-Dev/Services/Branches/Internet
#Check SSH from Hub2-Services to Hub2-Dev/Prod/Branches/Internet
#Check SSH from Hub2-Branches to Hub2-Dev/Prod/Services/Internet
```

### 3. Check if HTTP is blocked or allowed through FW
#### 3.1 From Hub1:
```
#check HTTP from Hub1-Dev to Hub1-Prod/Services/Branches/Internet
Example: curl 10.1.1.4 
#Check HTTP from Hub1-Prod to Hub1-Dev/Services/Branches/Internet
#Check HTTP from Hub1-Services to Hub1-Dev/Prod/Branches/Internet
#Check HTTP from Hub1-Branches to Hub1-Dev/Prod/Services/Internet

#check HTTP from Hub1-Dev to Hub2-Prod/Services/Branches/Internet
#Check HTTP from Hub1-Prod to Hub2-Dev/Services/Branches/Internet
#Check HTTP from Hub1-Services to Hub2-Dev/Prod/Branches/Internet
#Check HTTP from Hub1-Branches to Hub2-Dev/Prod/Services/Internet
```
#### 3.2 From Hub2:
```
#check HTTP from Hub2-Dev to Hub1-Prod/Services/Branches/Internet
#Check HTTP from Hub2-Prod to Hub1-Dev/Services/Branches/Internet
#Check HTTP from Hub2-Services to Hub1-Dev/Prod/Branches/Internet
#Check HTTP from Hub2-Branches to Hub1-Dev/Prod/Services/Internet

#check HTTP from Hub2-Dev to Hub2-Prod/Services/Branches/Internet
#Check HTTP from Hub2-Prod to Hub2-Dev/Services/Branches/Internet
#Check HTTP from Hub2-Services to Hub2-Dev/Prod/Branches/Internet
#Check HTTP from Hub2-Branches to Hub2-Dev/Prod/Services/Internet
```
