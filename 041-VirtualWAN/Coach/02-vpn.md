# Challenge 2. VPN - Coach's Guide

[< Previous Challenge](./01-any_to_any.md) - **[Home](./README.md)** - [Next Challenge >](./03-isolated_vnet.md)

## Notes and Guidance

- Cisco CSRs do not generate any additional cost
- These scripts leverage Cisco CSR as NVA. If preferred, you can use any other NVA to simulate an onprem VPN device, such as Windows Server, Linux StrongSwan or any other

## Solution Guide

## 1. Create VPN Gateways in the vHubs

Note that VPN creation can take some time:

```bash
# Create VPN gateways
az network vpn-gateway create -n hubvpn1 -g $rg -l $location1 --vhub hub1 --asn 65515
az network vpn-gateway create -n hubvpn2 -g $rg -l $location2 --vhub hub2 --asn 65515
```

## 2. Create Branch 1 and 2

Cisco CSRs will only cost the VM pricing:

### 2.1 Create CSR to simulate branch 1
```
az vm image terms accept --urn ${publisher}:${offer}:${sku}:${version}
az vm create -n branch1-nva -g $rg -l $location1 --image ${publisher}:${offer}:${sku}:${version} \
    --admin-username "$username" --admin-password "$password" --authentication-type all --generate-ssh-keys \
    --public-ip-address branch1-pip --public-ip-address-allocation static \
    --vnet-name branch1 --vnet-address-prefix $branch1_prefix --subnet nva --subnet-address-prefix $branch1_subnet \
    --private-ip-address $branch1_bgp_ip
branch1_ip=$(az network public-ip show -n branch1-pip -g $rg --query ipAddress -o tsv)
az network vpn-site create -n branch1 -g $rg -l $location1 --virtual-wan $vwan \
    --asn $branch1_asn --bgp-peering-address $branch1_bgp_ip --ip-address $branch1_ip --address-prefixes ${branch1_ip}/32 --device-vendor cisco --device-model csr --link-speed 100
az network vpn-gateway connection create -n branch1 --gateway-name hubvpn1 -g $rg --remote-vpn-site branch1 \
    --enable-bgp true --protocol-type IKEv2 --shared-key "$password" --connection-bandwidth 100 --routing-weight 10 \
    --associated-route-table $hub1_default_rt_id --propagated-route-tables $hub1_default_rt_id --labels default --internet-security true
```

### 2.2 Create CSR to simulate branch 2
```
az vm create -n branch2-nva -g $rg -l $location2 --image ${publisher}:${offer}:${sku}:${version} \
    --admin-username "$username" --admin-password "$password" --authentication-type all --generate-ssh-keys \
    --public-ip-address branch2-pip --public-ip-address-allocation static \
    --vnet-name branch2 --vnet-address-prefix $branch2_prefix --subnet nva --subnet-address-prefix $branch2_subnet \
    --private-ip-address $branch2_bgp_ip
branch2_ip=$(az network public-ip show -n branch2-pip -g $rg --query ipAddress -o tsv)
az network vpn-site create -n branch2 -g $rg -l $location2 --virtual-wan $vwan \
    --asn $branch2_asn --bgp-peering-address $branch2_bgp_ip --ip-address $branch2_ip --address-prefixes ${branch2_ip}/32
az network vpn-gateway connection create -n branch2 --gateway-name hubvpn2 -g $rg --remote-vpn-site branch2 \
    --enable-bgp true --protocol-type IKEv2 --shared-key "$password" --connection-bandwidth 100 --routing-weight 10 \
    --associated-route-table $hub2_default_rt_id --propagated-route-tables $hub2_default_rt_id  --labels default --internet-security true
```

## 3. Configure CSRs of Brnach 1 and 2

Configuring the CSRs will add the required IPsec and BGP configurations

### 3.1 Branch 1
#### 3.1.1 Generate VPN configuration 

```bash
# Get parameters for VPN GW in hub1
vpngw1_config=$(az network vpn-gateway show -n hubvpn1 -g $rg)
site=branch1
vpngw1_gw0_pip=$(echo $vpngw1_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]')
vpngw1_gw1_pip=$(echo $vpngw1_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].tunnelIpAddresses[0]')
vpngw1_gw0_bgp_ip=$(echo $vpngw1_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]')
vpngw1_gw1_bgp_ip=$(echo $vpngw1_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].defaultBgpIpAddresses[0]')
vpngw1_bgp_asn=$(echo $vpngw1_config | jq -r '.bgpSettings.asn')  # This is today always 65515
echo "Extracted info for hubvpn1: Gateway0 $vpngw1_gw0_pip, $vpngw1_gw0_bgp_ip. Gateway1 $vpngw1_gw1_pip, $vpngw1_gw0_bgp_ip. ASN $vpngw1_bgp_asn"

# Create CSR config for branch 1
csr_config_url="https://raw.githubusercontent.com/sada-kubsad/WhatTheHack/master/041-VirtualWAN/Coach/csr_config_2tunnels_tokenized.txt"
config_file_csr='branch1_csr.cfg'
config_file_local='./branch1_csr.cfg'
wget $csr_config_url -O $config_file_local
sed -i "s|\*\*PSK\*\*|${password}|g" $config_file_local
sed -i "s|\*\*GW0_Private_IP\*\*|${vpngw1_gw0_bgp_ip}|g" $config_file_local
sed -i "s|\*\*GW1_Private_IP\*\*|${vpngw1_gw1_bgp_ip}|g" $config_file_local
sed -i "s|\*\*GW0_Public_IP\*\*|${vpngw1_gw0_pip}|g" $config_file_local
sed -i "s|\*\*GW1_Public_IP\*\*|${vpngw1_gw1_pip}|g" $config_file_local
sed -i "s|\*\*BGP_ID\*\*|${branch1_asn}|g" $config_file_local
ssh -o BatchMode=yes -o StrictHostKeyChecking=no $username@$branch1_ip <<EOF
  config t
    file prompt quiet
EOF
scp $config_file_local ${branch1_ip}:/${config_file_csr}
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch1_ip "copy bootflash:${config_file_csr} running-config"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch1_ip "wr mem"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch1_ip "sh ip int b"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch1_ip "sh ip bgp summary"

```

The above commands turns [this config file](https://github.com/sada-kubsad/WhatTheHack/blob/master/041-VirtualWAN/Coach/csr_config_2tunnels_tokenized.txt) into the following config being applied to Branch 1 CSR: 
```
# Note: Before you can apply the configuration below, enable DNA licensing by running the below. This is required so that the crypto ikev2... command gets enabled: 

config t
license boot level network-essentials addon dna-essentials
exit
wr mem
reload         --> Reboot required for license to take effect


crypto ikev2 proposal azure-proposal
  encryption aes-cbc-256 aes-cbc-128
  integrity sha1 sha256 sha384 sha512 
  group 14 15 16 
  exit
!
crypto ikev2 policy azure-policy
  proposal azure-proposal
  exit
!
crypto ikev2 keyring azure-keyring
  peer 20.115.177.8
    address 20.115.177.8
    pre-shared-key Msft123Msft123
    exit
  peer 20.115.177.15
    address 20.115.177.15
    pre-shared-key Msft123Msft123
    exit
  exit
!
crypto ikev2 profile azure-profile
  match address local interface GigabitEthernet1
  match identity remote address 20.115.177.8 255.255.255.255
  match identity remote address 20.115.177.15 255.255.255.255
  authentication remote pre-share
  authentication local pre-share
  keyring local azure-keyring
  exit
!
crypto ipsec transform-set azure-ipsec-proposal-set esp-aes 256 esp-sha-hmac
 mode tunnel
 exit

crypto ipsec profile azure-vti
  set transform-set azure-ipsec-proposal-set
  set ikev2-profile azure-profile
  set security-association lifetime kilobytes 102400000
  set security-association lifetime seconds 3600
 exit
!
interface Tunnel0
 ip unnumbered GigabitEthernet1
 ip tcp adjust-mss 1350
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 20.115.177.8
 tunnel protection ipsec profile azure-vti
exit
!
interface Tunnel1
 ip unnumbered GigabitEthernet1
 ip tcp adjust-mss 1350
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 20.115.177.15
 tunnel protection ipsec profile azure-vti
exit

!
router bgp 65001
 bgp router-id interface GigabitEthernet1
 bgp log-neighbor-changes
 neighbor 192.168.1.13 remote-as 65515
 neighbor 192.168.1.13 ebgp-multihop 5
 neighbor 192.168.1.13 update-source GigabitEthernet1
 neighbor 192.168.1.12 remote-as 65515
 neighbor 192.168.1.12 ebgp-multihop 5
 neighbor 192.168.1.12 update-source GigabitEthernet1
!
ip route 192.168.1.13 255.255.255.255 Tunnel0
ip route 192.168.1.12 255.255.255.255 Tunnel1
!
end
!
wr mem
```
#### 3.1.2 Configure BGP:
Generate the BGP configuration for Branch 1 CSR with:
```
myip=$(curl -s4 ifconfig.co)
loopback_ip=10.11.11.11
default_gateway=$branch1_gateway
ssh -o BatchMode=yes -o StrictHostKeyChecking=no $branch1_ip <<EOF
config t
    no ip domain lookup
    interface Loopback0
        ip address ${loopback_ip} 255.255.255.255
    router bgp ${branch1_asn}
        redistribute connected
    ip route ${vpngw1_gw0_pip} 255.255.255.255 ${default_gateway}
    ip route ${vpngw1_gw1_pip} 255.255.255.255 ${default_gateway}
    ip route ${myip} 255.255.255.255 ${default_gateway}
    line vty 0 15
        exec-timeout 0 0
end
EOF
```
The above results in the following BGP configuration: 

```
config t
    no ip domain lookup                            --> no IP DNS hostname translation
    interface Loopback0
        ip address 10.11.11.11 255.255.255.255
    router bgp 65001
        redistribute connected
    ip route 20.115.177.8 255.255.255.255 172.16.1.1
    ip route 20.115.177.15 255.255.255.255 172.16.1.1
    ip route 172.251.40.139 255.255.255.255 172.16.1.1
    line vty 0 15                                --> Config Visual terminal line with first line number =0 & last
                                                    line number=15
        exec-timeout 0 0                        --> Set Exec Timeout 0 mins & 0 seconds
end
!
wr mem
```
## 3.2 Branch 2
#### 3.2.1 Generate VPN configurion 
```
# Get parameters for VPN GW in hub2
vpngw2_config=$(az network vpn-gateway show -n hubvpn2 -g $rg)
site=branch2
vpngw2_gw0_pip=$(echo $vpngw2_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].tunnelIpAddresses[0]')
vpngw2_gw1_pip=$(echo $vpngw2_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].tunnelIpAddresses[0]')
vpngw2_gw0_bgp_ip=$(echo $vpngw2_config | jq -r '.bgpSettings.bgpPeeringAddresses[0].defaultBgpIpAddresses[0]')
vpngw2_gw1_bgp_ip=$(echo $vpngw2_config | jq -r '.bgpSettings.bgpPeeringAddresses[1].defaultBgpIpAddresses[0]')
vpngw2_bgp_asn=$(echo $vpngw2_config | jq -r '.bgpSettings.asn')  # This is today always 65515
echo "Extracted info for hubvpn2: Gateway0 $vpngw2_gw0_pip, $vpngw2_gw0_bgp_ip. Gateway1 $vpngw2_gw1_pip, $vpngw2_gw0_bgp_ip. ASN $vpngw2_bgp_asn"

csr_config_url="https://raw.githubusercontent.com/sada-kubsad/WhatTheHack/master/041-VirtualWAN/Coach/csr_config_2tunnels_tokenized.txt"
config_file_csr='branch2_csr.cfg'
config_file_local='./branch2_csr.cfg'
wget $csr_config_url -O $config_file_local
sed -i "s|\*\*PSK\*\*|${password}|g" $config_file_local
sed -i "s|\*\*GW0_Private_IP\*\*|${vpngw2_gw0_bgp_ip}|g" $config_file_local
sed -i "s|\*\*GW1_Private_IP\*\*|${vpngw2_gw1_bgp_ip}|g" $config_file_local
sed -i "s|\*\*GW0_Public_IP\*\*|${vpngw2_gw0_pip}|g" $config_file_local
sed -i "s|\*\*GW1_Public_IP\*\*|${vpngw2_gw1_pip}|g" $config_file_local
sed -i "s|\*\*BGP_ID\*\*|${branch2_asn}|g" $config_file_local
ssh -o BatchMode=yes -o StrictHostKeyChecking=no $username@$branch2_ip <<EOF
  config t
    file prompt quiet
EOF
scp $config_file_local ${branch2_ip}:/${config_file_csr}
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch2_ip "copy bootflash:${config_file_csr} running-config"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch2_ip "wr mem"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch2_ip "sh ip int b"
ssh -n -o BatchMode=yes -o StrictHostKeyChecking=no $branch2_ip "sh ip bgp summary"

```

The above commands turn [this config file](https://github.com/sada-kubsad/WhatTheHack/blob/master/041-VirtualWAN/Coach/csr_config_2tunnels_tokenized.txt) into the following config being applied to CSR for Hub 2: 

```
# Before you can apply the configuration below, enable DNA licensing by running the below. This is required so that the crypto ikev2... command gets enabled: 

config t
license boot level network-essentials addon dna-essentials
exit
wr mem
reload         --> Reboot required for license to take effect


crypto ikev2 proposal azure-proposal
  encryption aes-cbc-256 aes-cbc-128
  integrity sha1 sha256 sha384 sha512 
  group 14 15 16 
  exit
!
crypto ikev2 policy azure-policy
  proposal azure-proposal
  exit
!
crypto ikev2 keyring azure-keyring
  peer 4.152.26.209
    address 4.152.26.209
    pre-shared-key Msft123Msft123
    exit
  peer 4.152.29.218
    address 4.152.29.218
    pre-shared-key Msft123Msft123
    exit
  exit
!
crypto ikev2 profile azure-profile
  match address local interface GigabitEthernet1
  match identity remote address 4.152.26.209 255.255.255.255
  match identity remote address 4.152.29.218 255.255.255.255
  authentication remote pre-share
  authentication local pre-share
  keyring local azure-keyring
  exit
!
crypto ipsec transform-set azure-ipsec-proposal-set esp-aes 256 esp-sha-hmac
 mode tunnel
 exit

crypto ipsec profile azure-vti
  set transform-set azure-ipsec-proposal-set
  set ikev2-profile azure-profile
  set security-association lifetime kilobytes 102400000
  set security-association lifetime seconds 3600
 exit
!
interface Tunnel0
 ip unnumbered GigabitEthernet1
 ip tcp adjust-mss 1350
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 4.152.26.209
 tunnel protection ipsec profile azure-vti
exit
!
interface Tunnel1
 ip unnumbered GigabitEthernet1
 ip tcp adjust-mss 1350
 tunnel source GigabitEthernet1
 tunnel mode ipsec ipv4
 tunnel destination 4.152.29.218
 tunnel protection ipsec profile azure-vti
exit

!
router bgp 65002
 bgp router-id interface GigabitEthernet1
 bgp log-neighbor-changes
 neighbor 192.168.2.12 remote-as 65515
 neighbor 192.168.2.12 ebgp-multihop 5
 neighbor 192.168.2.12 update-source GigabitEthernet1
 neighbor 192.168.2.13 remote-as 65515
 neighbor 192.168.2.13 ebgp-multihop 5
 neighbor 192.168.2.13 update-source GigabitEthernet1
!
ip route 192.168.2.12 255.255.255.255 Tunnel0
ip route 192.168.2.13 255.255.255.255 Tunnel1
!
end
!
wr mem
```
#### 3.2.2 Configure BGP:
Generate BGP configuration for Branch 2 CSR with:
```
myip=$(curl -s4 ifconfig.co)
loopback_ip=10.22.22.22
default_gateway=$branch2_gateway
ssh -o BatchMode=yes -o StrictHostKeyChecking=no $branch2_ip <<EOF
config t
    no ip domain lookup
    interface Loopback0
        ip address ${loopback_ip} 255.255.255.255
    router bgp ${branch2_asn}
        redistribute connected
    ip route ${vpngw2_gw0_pip} 255.255.255.255 ${default_gateway}
    ip route ${vpngw2_gw1_pip} 255.255.255.255 ${default_gateway}
    ip route ${myip} 255.255.255.255 ${default_gateway}
    line vty 0 15
        exec-timeout 0 0
end
EOF
```
The above results in the following BGP configuration: 
```
config t
    no ip domain lookup
    interface Loopback0
        ip address 10.22.22.22 255.255.255.255
    router bgp 65002
        redistribute connected
    ip route 4.152.26.209 255.255.255.255 172.16.2.1
    ip route 4.152.29.218 255.255.255.255 172.16.2.1
    ip route 172.251.40.139 255.255.255.255 172.16.2.1
    line vty 0 15
        exec-timeout 0 0
end
! wr mem
```
## 4. Verify that all tunnels are up, and BGP adjacencies established:
Initially both Tunnels at Branch 1 and 2 were down:
```bash
#
show ip interface brief"
branch1-nva#show ip interface brief
# Tunnels not setup:
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       172.16.1.10     YES DHCP   up                    up
Loopback0              10.11.11.11     YES NVRAM  up                    up
Tunnel0                172.16.1.10     YES TFTP   up                    down
Tunnel1                172.16.1.10     YES TFTP   up                    down
VirtualPortGroup0      192.168.35.101  YES NVRAM  up                    up

# Tunnels correctly setup:
Interface              IP-Address      OK? Method Status                Protocol
GigabitEthernet1       172.16.1.10     YES DHCP   up                    up
Loopback0              10.11.11.11     YES NVRAM  up                    up
Tunnel0                172.16.1.10     YES TFTP   up                    up
Tunnel1                172.16.1.10     YES TFTP   up                    up
VirtualPortGroup0      192.168.35.101  YES NVRAM  up                    up

#
show ip bgp summary
branch1-nva#show ip bgp summary

# BGP NOT correctly setup:
BGP router identifier 10.11.11.11, local AS number 65001
BGP table version is 3, main routing table version 3
2 network entries using 496 bytes of memory
2 path entries using 272 bytes of memory
1/1 BGP path/bestpath attribute entries using 296 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 1064 total bytes of memory
BGP activity 2/0 prefixes, 2/0 paths, scan interval 60 secs
2 networks peaked at 15:29:21 May 23 2024 UTC (01:06:24.217 ago)

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ         Up/Down      State/PfxRcd
192.168.1.12    4        65515       0       0        1    0    0         never        Idle
192.168.1.13    4        65515       0       0        1    0    0         never        Idle

# BGP correctly setup:
BGP router identifier 10.11.11.11, local AS number 65001
BGP table version is 3, main routing table version 3
2 network entries using 496 bytes of memory
2 path entries using 272 bytes of memory
1/1 BGP path/bestpath attribute entries using 296 bytes of memory
0 BGP route-map cache entries using 0 bytes of memory
0 BGP filter-list cache entries using 0 bytes of memory
BGP using 1064 total bytes of memory
BGP activity 2/0 prefixes, 2/0 paths, scan interval 60 secs
2 networks peaked at 15:29:21 May 23 2024 UTC (01:06:24.217 ago)

Neighbor        V           AS MsgRcvd MsgSent   TblVer  InQ OutQ         Up/Down          State/PfxRcd
192.168.1.12    4        65515       3       6        1    0    0         00:00:31         3
192.168.1.13    4        65515       3       6        1    0    0         00:00:31         3

```
## 5. Troubleshoot VPN+BGP setup
### 5.1 Troubleshoot Crypto session
```
show crypto ikev2 session
# Healthy output:
IPv4 Crypto IKEv2 Session

Session-id:1, Status:UP-ACTIVE, IKE count:1, CHILD count:1

Tunnel-id Local Remote fvrf/ivrf Status 
1 10.4.5.224/500 10.4.5.225/500 none/90 READY 
Encr: AES-CBC, keysize: 128, PRF: SHA1, Hash: SHA96, DH Grp:14, Auth sign: PSK, Auth verify: PSK
Life/Active Time: 86400/207 sec
Child sa: local selector 0.0.0.0/0 - 255.255.255.255/65535
remote selector 0.0.0.0/0 - 255.255.255.255/65535
ESP spi in/out: 0xFC13A6B7/0x1A2AC4A0

# Not healthly output:
<blank>
```

```
show crypto ipsec sa sa interface Tunnel<id>

# Healthy output:

# Not Healthy ouput:
interface: Tunnel0
    Crypto map tag: Tunnel0-head-0, local addr 172.16.1.10

   protected vrf: (none)
   local  ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   remote ident (addr/mask/prot/port): (0.0.0.0/0.0.0.0/0/0)
   current_peer 20.115.177.8 port 500
     PERMIT, flags={origin_is_acl,}
    #pkts encaps: 0, #pkts encrypt: 0, #pkts digest: 0
    #pkts decaps: 0, #pkts decrypt: 0, #pkts verify: 0
    #pkts compressed: 0, #pkts decompressed: 0
    #pkts not compressed: 0, #pkts compr. failed: 0
    #pkts not decompressed: 0, #pkts decompress failed: 0
    #send errors 0, #recv errors 0

     local crypto endpt.: 172.16.1.10, remote crypto endpt.: 20.115.177.8
     plaintext mtu 1500, path mtu 1500, ip mtu 1500, ip mtu idb GigabitEthernet1
     current outbound spi: 0x0(0)
     PFS (Y/N): N, DH group: none

     inbound esp sas:

     inbound ah sas:

     inbound pcp sas:

     outbound esp sas:

     outbound ah sas:

     outbound pcp sas:

show crypto ikev2 sa
```
#### Sample output of not established tunnel:
![image](https://github.com/sada-kubsad/WhatTheHack/assets/11302503/6709ed02-e2c4-46cd-995a-38ff417268be)

##### Sample output of established tunnel:
![image](https://github.com/sada-kubsad/WhatTheHack/assets/11302503/602aaf79-ed04-4b63-b74d-273392f5571b)

### 5.2 See Ikev2 info 
```
show crypto ikev2 stats 
```
#### Sample output:

![image](https://github.com/sada-kubsad/WhatTheHack/assets/11302503/aedca6b6-c97b-476a-8c60-54ddc0ceaeaa)


### 5.3 Check logs:
```
show logging  | include Tunnel

#Result when down:
*May 30 16:57:21.189: %LINEPROTO-5-UPDOWN: Line protocol on Interface Tunnel0, changed state to down
*May 30 16:57:32.232: %LINEPROTO-5-UPDOWN: Line protocol on Interface Tunnel1, changed state to down

#Reuslt when normal:

```

### 5.4 Check Tunnnel status:
```
show crypto ikev2 sa
```

### 5.5 Check ping connectivity
```
#From the Branch 1 CSR console
ping 192.168.1.12
ping 192.168.1.13

#From the Branch 2 CSR console
ping 192.168.2.12
ping 192.168.2.13

```

## 6. Start, Stop and Check status of VMs
```
Start All VMs:
--------------
az vm start  -g wthvwan -n spoke11-vm   > /dev/null 2>&1 & 
az vm start  -g wthvwan -n spoke12-vm   > /dev/null 2>&1 & 
az vm start  -g wthvwan -n spoke21-vm   > /dev/null 2>&1 & 
az vm start  -g wthvwan -n spoke22-vm   > /dev/null 2>&1 &

az vm start  -g wthvwan -n branch1-nva   > /dev/null 2>&1 &
az vm start  -g wthvwan -n branch2-nva   > /dev/null 2>&1 &
 

Stop Deallocate All VMs:
-------------------------
az vm deallocate -g wthvwan -n spoke11-vm  > /dev/null 2>&1 & 
az vm deallocate -g wthvwan -n spoke12-vm  > /dev/null 2>&1 & 
az vm deallocate -g wthvwan -n spoke21-vm  > /dev/null 2>&1 & 
az vm deallocate -g wthvwan -n spoke22-vm  > /dev/null 2>&1 & 

az vm deallocate -g wthvwan -n branch1-nva  > /dev/null 2>&1 & 
az vm deallocate -g wthvwan -n branch2-nva  > /dev/null 2>&1 & 


Check the status of VMSs:
-------------------------
az vm get-instance-view -g wthvwan -n spoke11-vm    | grep -i power
az vm get-instance-view -g wthvwan -n spoke12-vm    | grep -i power
az vm get-instance-view -g wthvwan -n spoke21-vm    | grep -i power
az vm get-instance-view -g wthvwan -n spoke22-vm    | grep -i power

az vm get-instance-view -g wthvwan -n branch1-nva   | grep -i power
az vm get-instance-view -g wthvwan -n branch2-nva   | grep -i power

```

## Connect to VMs
```
export spoke11vm=$(az network public-ip show -n spoke11-pip -g wthvwan --query ipAddress -o tsv)
export spoke12vm=$(az network public-ip show -n spoke12-pip -g wthvwan --query ipAddress -o tsv)
export spoke21vm=$(az network public-ip show -n spoke21-pip -g wthvwan --query ipAddress -o tsv)
export spoke22vm=$(az network public-ip show -n spoke22-pip -g wthvwan --query ipAddress -o tsv)



export branch1_ip=$(az network public-ip show -n branch1-pip -g wthvwan --query ipAddress -o tsv)
export branch2_ip=$(az network public-ip show -n branch2-pip -g wthvwan --query ipAddress -o tsv)


ssh azureuser@$branch1_ip
ssh azureuser@$branch2_ip
```
