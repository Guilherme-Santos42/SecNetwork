# SecNetwork (Secure Network LAB)
A network with a DMZ, VPN, Hardening, and ACLs.

Scheme:<br>

<img width="489" height="434" alt="image" src="https://github.com/user-attachments/assets/c3e60e7c-6699-41f4-9931-a1e595a7d446" />

Equipment configuration:

1- Global Preparation (License Activation)
Router(config)# license boot module c2900 technology-package securityk9
Router(config)# exit
Router# copy running-config startup-config
Router# reload

Detailed Command Explanation:

license boot module c2900 technology-package securityk9: Commands the router to load the security feature set (including VPN/Encryption) upon the next boot.
copy running-config startup-config: Saves the license change to NVRAM.
reload: Restarts the router to initialize the newly activated security features.

*This activates the securityk9 license. Without this, the crypto command will be rejected by the CLI.

2- Router 3 Configuration (HQ & DMZ)
This router manages the Internal LAN, the DMZ (Public Server), and the "Headquarters" side of the VPN.

Interfaces and Routing:

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

Detailed Command Explanation:

interface g0/x: Enters configuration mode for specific physical ports.

description: Adds a label for administrative clarity.

ip address [IP] [Mask]: Assigns the logical identity to the interface.

ip route 192.168.20.0 ... 10.0.0.2: A static route telling HQ that any traffic for the Branch LAN (20.0) must be sent to the Branch Router’s WAN IP (10.0.0.2).

DMZ Security (ACLs):

Router3(config)# ip access-list extended FW_DMZ
Router3(config-ext-nacl)# permit tcp any host 172.16.0.10 eq 80
Router3(config-ext-nacl)# deny ip 172.16.0.0 0.0.0.255 192.168.10.0 0.0.0.255
Router3(config-ext-nacl)# permit ip any any
Router3(config)# interface g0/1
Router3(config-if)# ip access-group FW_DMZ in

*This permits HTTP traffic to the server but strictly blocks the DMZ from initiating any connection to the internal LAN (Preventing lateral movement).

Detailed Command Explanation:

ip access-list extended FW_DMZ: Creates a named list of rules to filter traffic based on source, destination, and port.
permit tcp any host ... eq 80: Allows web traffic (HTTP) from anywhere to reach the specific Server1 IP.
deny ip 172.16.0.0 ... 192.168.10.0 ...: Blocks the DMZ network from initiating any connection to the Internal LAN. This prevents "lateral movement" if the web server is compromised.
ip access-group FW_DMZ in: Applies the rules to traffic entering the router from the DMZ interface.

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

On Router 3 (HQ) :

Phase 1: ISAKMP (IKE) 
Router3(config)# crypto isakmp policy 10
Router3(config-isakmp)# encryption aes
Router3(config-isakmp)# hash sha
Router3(config-isakmp)# authentication pre-share
Router3(config-isakmp)# group 2
Router3(config)# crypto isakmp key cisco123 address 10.0.0.2

Phase 2: IPsec - The Data Tunnel
Router3(config)# crypto ipsec transform-set ESP_SET esp-aes esp-sha-hmac
Router3(config)# access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255

Phase 3: Crypto Map - The Application
Router3(config)# crypto map VPN_MAP 10 ipsec-isakmp
Router3(config-crypto-map)# set peer 10.0.0.2
Router3(config-crypto-map)# set transform-set ESP_SET
Router3(config-crypto-map)# match address 110
Router3(config)# interface g0/0
Router3(config-if)# crypto map VPN_MAP

Explanation: Both routers must agree on these parameters (aes for encryption, sha for integrity, group 2 for key exchange). The key is the shared password.

On Router 2 (Branch):

Router2(config)# crypto isakmp policy 10
Router2(config-isakmp)# encryption aes
Router2(config-isakmp)# hash sha
Router2(config-isakmp)# authentication pre-share
Router2(config-isakmp)# group 2
Router2(config)# crypto isakmp key cisco123 address 10.0.0.1

Router2(config)# crypto ipsec transform-set ESP_SET esp-aes esp-sha-hmac
Router2(config)# access-list  permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255

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

Detailed Command Explanation:
switchport port-security: Enables the security engine on the port.
maximum 1: Only allows one device to connect to this port.
violation shutdown: If a second MAC address is detected, the port is immediately disabled.
mac-address sticky: The switch "remembers" the first MAC address it sees and writes it into the running configuration.

B. Management Hardening (SSH): Disable Telnet and enable encrypted SSH access.

(config)# ip domain-name securelab.com
(config)# crypto key generate rsa
! Choose 1024
(config)# username admin privilege 15 secret cisco123
(config)# line vty 0 4
(config-line)# login local
(config-line)# transport input ssh

Detailed Command Explanation:
crypto key generate rsa: Generates the encryption keys needed for SSH.
line vty 0 4: Configures the virtual terminal lines (remote access).
transport input ssh: This is the critical security step—it explicitly forbids Telnet and only permits SSH.

6- Endpoint Addressing (Manual Configuration):
Device,IP Address,Subnet Mask,Default Gateway(RESPECTIVELY)
PC2 (HQ LAN),192.168.10.10,255.255.255.0,192.168.10.1
Server1 (DMZ),172.16.0.10,255.255.255.0,172.16.0.1
PC3 (Branch LAN),192.168.20.10,255.255.255.0,192.168.20.1

TESTING:

VPN: Ping from PC2 to PC3. Then run show crypto isakmp sa. Status should be QM_IDLE.
DMZ: Ping from Server1 to PC2. It should fail (Blocked by ACL).
Hardening: Unplug PC2 and plug in a different PC. The switch port should turn red (Shutdown).


No mundo ideal a configuração perfeita seria se adicionassemos firewalls de próxima geração (NGFW), nos lugares dos roteadores, no entanto no packet tracer possuimos certas limitações. Seria interessante também adicionarmos um servidor RADIUS/AAA e também um IDS, ou SIEM para monitoramento de alertas gerados na rede, No entanto encontrei problemas para configurar o servidor SNMP e syslog.
Ficaria mais ou menos assim ( Sem servidor AAA):
<img width="817" height="553" alt="image" src="https://github.com/user-attachments/assets/022feaf1-c513-4b0c-93d3-fb60c78deb8f" />

Configuração FW 1 :
# 1. Definir o tráfego que deve passar pela VPN (Interessante)
ciscoasa(config)# access-list VPN_ACL extended permit ip 192.168.10.0 255.255.255.0 192.168.20.0 255.255.255.0

# 2. Fase 1 - IKEv1 Policy
ciscoasa(config)# crypto ikev1 policy 10
ciscoasa(config-ikev1-policy)# encryption aes
ciscoasa(config-ikev1-policy)# hash sha
ciscoasa(config-ikev1-policy)# authentication pre-share
ciscoasa(config-ikev1-policy)# group 2
ciscoasa(config-ikev1-policy)# lifetime 86400

# 3. Fase 2 - Transform Set
ciscoasa(config)# crypto ipsec ikev1 transform-set ESP_SET esp-aes esp-sha-hmac

# 4. Tunnel Group (Onde definimos a senha/peer)
ciscoasa(config)# tunnel-group 10.0.0.2 type ipsec-l2l
ciscoasa(config)# tunnel-group 10.0.0.2 ipsec-attributes
ciscoasa(config-tunnel-ipsec)# ikev1 pre-shared-key cisco123

# 5. Crypto Map e Ativação
ciscoasa(config)# crypto map MY_MAP 10 match address VPN_ACL
ciscoasa(config)# crypto map MY_MAP 10 set peer 10.0.0.2
ciscoasa(config)# crypto map MY_MAP 10 set ikev1 transform-set ESP_SET
ciscoasa(config)# crypto map MY_MAP interface outside
ciscoasa(config)# crypto ikev1 enable outside

Configuração FW 2:
# Configuração de Interface
ciscoasa(config)# interface g1/1
ciscoasa(config-if)# nameif outside
ciscoasa(config-if)# ip address 10.0.0.2 255.255.255.252
ciscoasa(config-if)# no shutdown

ciscoasa(config)# interface g1/2
ciscoasa(config-if)# nameif inside
ciscoasa(config-if)# ip address 192.168.20.1 255.255.255.0
ciscoasa(config-if)# no shutdown

# Rota para chegar na HQ
ciscoasa(config)# route outside 192.168.10.0 255.255.255.0 10.0.0.1

# VPN (Espelhada)
ciscoasa(config)# access-list VPN_ACL extended permit ip 192.168.20.0 255.255.255.0 192.168.10.0 255.255.255.0
ciscoasa(config)# crypto ikev1 policy 10
ciscoasa(config-ikev1-policy)# encryption aes
ciscoasa(config-ikev1-policy)# hash sha
ciscoasa(config-ikev1-policy)# authentication pre-share
ciscoasa(config-ikev1-policy)# group 2
ciscoasa(config)# tunnel-group 10.0.0.1 type ipsec-l2l
ciscoasa(config)# tunnel-group 10.0.0.1 ipsec-attributes
ciscoasa(config-tunnel-ipsec)# ikev1 pre-shared-key cisco123
ciscoasa(config)# crypto ipsec ikev1 transform-set ESP_SET esp-aes esp-sha-hmac
ciscoasa(config)# crypto map MY_MAP 10 match address VPN_ACL
ciscoasa(config)# crypto map MY_MAP 10 set peer 10.0.0.1
ciscoasa(config)# crypto map MY_MAP 10 set ikev1 transform-set ESP_SET
ciscoasa(config)# crypto map MY_MAP interface outside
ciscoasa(config)# crypto ikev1 enable outside

Criei também um "SIEM", mas ao invés de usar SNMP ou syslog usei uma SPAN port no local, redirecionando o trafego pro dispositivo:

Configuração no SWITCH da LAN:
Switch(config)# monitor session 1 source interface g0/1
Switch(config)# monitor session 1 destination interface g0/2
