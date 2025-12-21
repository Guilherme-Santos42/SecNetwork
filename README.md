# SecNetwork (Secure Network LAB)
A network with a DMZ, VPN, Hardening, and ACLs.

Scheme:
<img width="489" height="434" alt="image" src="https://github.com/user-attachments/assets/c3e60e7c-6699-41f4-9931-a1e595a7d446" />

Equipment configuration:

1- Global Preparation (License Activation)
Router(config)# license boot module c2900 technology-package securityk9
Router(config)# exit
Router# copy running-config startup-config
Router# reload

*This activates the securityk9 license. Without this, the crypto command will be rejected by the CLI.

2- Router 3 Configuration (HQ & DMZ)
This router manages the Internal LAN, the DMZ (Public Server), and the "Headquarters" side of the VPN.

1. Interfaces and Routing:

Router3(config)# interface g0/0
Router3(config-if)# description Link to Branch
Router3(config-if)# ip address 10.0.0.1 255.255.255.252
Router3(config-if)# no shutdown

Router3(config)# interface g0/1
Router3(config-if)# description DMZ_Interface
Router3(config-if)# ip address 172.16.0.1 255.255.255.0
Router3(config-if)# no shutdown

Router3(config)# interface g0/2
Router3(config-if)# description Internal_LAN
Router3(config-if)# ip address 192.168.10.1 255.255.255.0
Router3(config-if)# no shutdown

Router3(config)# ip route 192.168.20.0 255.255.255.0 10.0.0.2

DMZ Security (ACLs):

Router3(config)# ip access-list extended FW_DMZ
Router3(config-ext-nacl)# permit tcp any host 172.16.0.10 eq 80
Router3(config-ext-nacl)# deny ip 172.16.0.0 0.0.0.255 192.168.10.0 0.0.0.255
Router3(config-ext-nacl)# permit ip any any
Router3(config)# interface g0/1
Router3(config-if)# ip access-group FW_DMZ in

*This permits HTTP traffic to the server but strictly blocks the DMZ from initiating any connection to the internal LAN (Preventing lateral movement).

3- Router 2 Configuration (Branch)

Router2(config)# interface g0/0
Router2(config-if)# ip address 10.0.0.2 255.255.255.252
Router2(config-if)# no shutdown

Router2(config)# interface g0/1
Router2(config-if)# ip address 192.168.20.1 255.255.255.0
Router2(config-if)# no shutdown

Router2(config)# ip route 192.168.10.0 255.255.255.0 10.0.0.1

4- Site-to-Site VPN Implementation (IPsec)

The configuration must be mirrored between the two routers.

On Router 3 (HQ):

Router3(config)# crypto isakmp policy 10
Router3(config-isakmp)# encryption aes
Router3(config-isakmp)# hash sha
Router3(config-isakmp)# authentication pre-share
Router3(config-isakmp)# group 2
Router3(config)# crypto isakmp key cisco123 address 10.0.0.2

Router3(config)# crypto ipsec transform-set ESP_SET esp-aes esp-sha-hmac
Router3(config)# access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255

Router3(config)# crypto map VPN_MAP 10 ipsec-isakmp
Router3(config-crypto-map)# set peer 10.0.0.2
Router3(config-crypto-map)# set transform-set ESP_SET
Router3(config-crypto-map)# match address 110
Router3(config)# interface g0/0
Router3(config-if)# crypto map VPN_MAP

On Router 2 (Branch):

Router2(config)# crypto isakmp policy 10
Router2(config-isakmp)# encryption aes
Router2(config-isakmp)# hash sha
Router2(config-isakmp)# authentication pre-share
Router2(config-isakmp)# group 2
Router2(config)# crypto isakmp key cisco123 address 10.0.0.1

Router2(config)# crypto ipsec transform-set ESP_SET esp-aes esp-sha-hmac
Router2(config)# access-list 110 permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255

Router2(config)# crypto map VPN_MAP 10 ipsec-isakmp
Router2(config-crypto-map)# set peer 10.0.0.1
Router2(config-crypto-map)# set transform-set ESP_SET
Router2(config-crypto-map)# match address 110
Router2(config)# interface g0/0
Router2(config-if)# crypto map VPN_MAP

5- Device Hardening (L2 & L3)
A. Switch Hardening (Layer 2): Apply to all switches to prevent unauthorized physical access.

Switch(config)# interface range fa0/1-24
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport port-security
Switch(config-if-range)# switchport port-security maximum 1
Switch(config-if-range)# switchport port-security violation shutdown
Switch(config-if-range)# switchport port-security mac-address sticky

B. Management Hardening (SSH): Disable Telnet and enable encrypted SSH access.

(config)# ip domain-name securelab.com
(config)# crypto key generate rsa
! Choose 1024
(config)# username admin privilege 15 secret cisco123
(config)# line vty 0 4
(config-line)# login local
(config-line)# transport input ssh

6- Endpoint Addressing (Manual Configuration):
Device,IP Address,Subnet Mask,Default Gateway(RESPECTIVELY)
PC2 (HQ LAN),192.168.10.10,255.255.255.0,192.168.10.1
Server1 (DMZ),172.16.0.10,255.255.255.0,172.16.0.1
PC3 (Branch LAN),192.168.20.10,255.255.255.0,192.168.20.1

TESTING:

VPN: Ping from PC2 to PC3. Then run show crypto isakmp sa. Status should be QM_IDLE.
DMZ: Ping from Server1 to PC2. It should fail (Blocked by ACL).
Hardening: Unplug PC2 and plug in a different PC. The switch port should turn red (Shutdown).
