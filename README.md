
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Markdown Document</title>
    <style>
        body {
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Helvetica, Arial, sans-serif;
            line-height: 1.6;
            color: #333;
            max-width: 100%;
			height:1200;
            margin: 0 auto;
            padding: 2rem;
        }
        h1, h2, h3, h4, h5, h6 { margin-top: 1.5rem; margin-bottom: 1rem; }
        pre { background: #f6f8fa; padding: 1rem; border-radius: 6px; overflow-x: auto; }
        code { background: #f6f8fa; padding: 0.2rem 0.4rem; border-radius: 3px; }
        blockquote { border-left: 4px solid #dfe2e5; padding-left: 1rem; margin: 1rem 0; color: #6a737d; }
        table { border-collapse: collapse; width: 100%; margin: 1rem 0; }
        th, td { border: 1px solid #dfe2e5; padding: 0.5rem; text-align: left; }
        th { background: #f6f8fa; }
    </style>
</head>
<body>
<h1 id="h1-secnetwork-secure-network-lab-"><a name="SecNetwork (Secure Network LAB)" class="reference-link"></a><span class="header-link octicon octicon-link"></span>SecNetwork (Secure Network LAB)</h1><p>A network with a DMZ, VPN, Hardening, and ACLs.</p>
<p>Scheme:&lt;br&gt;</p>
<p>&lt;img width="489" height="434" alt="image" src="https://github.com/user-attachments/assets/c3e60e7c-6699-41f4-9931-a1e595a7d446" /&gt;</p>
<p>Equipment configuration:</p>
<p>1- Global Preparation (License Activation)<br>Router(config)# license boot module c2900 technology-package securityk9<br>Router(config)# exit<br>Router# copy running-config startup-config<br>Router# reload</p>
<p>Detailed Command Explanation:</p>
<p>license boot module c2900 technology-package securityk9: Commands the router to load the security feature set (including VPN/Encryption) upon the next boot.<br>copy running-config startup-config: Saves the license change to NVRAM.<br>reload: Restarts the router to initialize the newly activated security features.</p>
<p>*This activates the securityk9 license. Without this, the crypto command will be rejected by the CLI.</p>
<p>2- Router 3 Configuration (HQ &amp; DMZ)<br>This router manages the Internal LAN, the DMZ (Public Server), and the “Headquarters” side of the VPN.</p>
<p>Interfaces and Routing:</p>
<p>Router3(config)# interface g0/0<br>Router3(config-if)# description Link to Branch<br>Router3(config-if)# ip address 10.0.0.1 255.255.255.252<br>Router3(config-if)# no shutdown</p>
<p>Router3(config)# interface g0/1<br>Router3(config-if)# description DMZ_Interface<br>Router3(config-if)# ip address 172.16.0.1 255.255.255.0<br>Router3(config-if)# no shutdown</p>
<p>Router3(config)# interface g0/2<br>Router3(config-if)# description Internal_LAN<br>Router3(config-if)# ip address 192.168.10.1 255.255.255.0<br>Router3(config-if)# no shutdown</p>
<p>Router3(config)# ip route 192.168.20.0 255.255.255.0 10.0.0.2</p>
<p>Detailed Command Explanation:</p>
<p>interface g0/x: Enters configuration mode for specific physical ports.</p>
<p>description: Adds a label for administrative clarity.</p>
<p>ip address [IP] [Mask]: Assigns the logical identity to the interface.</p>
<p>ip route 192.168.20.0 … 10.0.0.2: A static route telling HQ that any traffic for the Branch LAN (20.0) must be sent to the Branch Router’s WAN IP (10.0.0.2).</p>
<p>DMZ Security (ACLs):</p>
<p>Router3(config)# ip access-list extended FW_DMZ<br>Router3(config-ext-nacl)# permit tcp any host 172.16.0.10 eq 80<br>Router3(config-ext-nacl)# deny ip 172.16.0.0 0.0.0.255 192.168.10.0 0.0.0.255<br>Router3(config-ext-nacl)# permit ip any any<br>Router3(config)# interface g0/1<br>Router3(config-if)# ip access-group FW_DMZ in</p>
<p>*This permits HTTP traffic to the server but strictly blocks the DMZ from initiating any connection to the internal LAN (Preventing lateral movement).</p>
<p>Detailed Command Explanation:</p>
<p>ip access-list extended FW_DMZ: Creates a named list of rules to filter traffic based on source, destination, and port.<br>permit tcp any host … eq 80: Allows web traffic (HTTP) from anywhere to reach the specific Server1 IP.<br>deny ip 172.16.0.0 … 192.168.10.0 …: Blocks the DMZ network from initiating any connection to the Internal LAN. This prevents “lateral movement” if the web server is compromised.<br>ip access-group FW_DMZ in: Applies the rules to traffic entering the router from the DMZ interface.</p>
<p>3- Router 2 Configuration (Branch)</p>
<p>Router2(config)# interface g0/0<br>Router2(config-if)# ip address 10.0.0.2 255.255.255.252<br>Router2(config-if)# no shutdown</p>
<p>Router2(config)# interface g0/1<br>Router2(config-if)# ip address 192.168.20.1 255.255.255.0<br>Router2(config-if)# no shutdown</p>
<p>Router2(config)# ip route 192.168.10.0 255.255.255.0 10.0.0.1</p>
<p>4- Site-to-Site VPN Implementation (IPsec)</p>
<p>The configuration must be mirrored between the two routers.</p>
<p>On Router 3 (HQ) :</p>
<p>Phase 1: ISAKMP (IKE)<br>Router3(config)# crypto isakmp policy 10<br>Router3(config-isakmp)# encryption aes<br>Router3(config-isakmp)# hash sha<br>Router3(config-isakmp)# authentication pre-share<br>Router3(config-isakmp)# group 2<br>Router3(config)# crypto isakmp key cisco123 address 10.0.0.2</p>
<p>Phase 2: IPsec - The Data Tunnel<br>Router3(config)# crypto ipsec transform-set ESP_SET esp-aes esp-sha-hmac<br>Router3(config)# access-list 110 permit ip 192.168.10.0 0.0.0.255 192.168.20.0 0.0.0.255</p>
<p>Phase 3: Crypto Map - The Application<br>Router3(config)# crypto map VPN_MAP 10 ipsec-isakmp<br>Router3(config-crypto-map)# set peer 10.0.0.2<br>Router3(config-crypto-map)# set transform-set ESP_SET<br>Router3(config-crypto-map)# match address 110<br>Router3(config)# interface g0/0<br>Router3(config-if)# crypto map VPN_MAP</p>
<p>Explanation: Both routers must agree on these parameters (aes for encryption, sha for integrity, group 2 for key exchange). The key is the shared password.</p>
<p>On Router 2 (Branch):</p>
<p>Router2(config)# crypto isakmp policy 10<br>Router2(config-isakmp)# encryption aes<br>Router2(config-isakmp)# hash sha<br>Router2(config-isakmp)# authentication pre-share<br>Router2(config-isakmp)# group 2<br>Router2(config)# crypto isakmp key cisco123 address 10.0.0.1</p>
<p>Router2(config)# crypto ipsec transform-set ESP_SET esp-aes esp-sha-hmac<br>Router2(config)# access-list  permit ip 192.168.20.0 0.0.0.255 192.168.10.0 0.0.0.255</p>
<p>Router2(config)# crypto map VPN_MAP 10 ipsec-isakmp<br>Router2(config-crypto-map)# set peer 10.0.0.1<br>Router2(config-crypto-map)# set transform-set ESP_SET<br>Router2(config-crypto-map)# match address 110<br>Router2(config)# interface g0/0<br>Router2(config-if)# crypto map VPN_MAP</p>
<p>5- Device Hardening (L2 &amp; L3)<br>A. Switch Hardening (Layer 2): Apply to all switches to prevent unauthorized physical access.</p>
<p>Switch(config)# interface range fa0/1-24<br>Switch(config-if-range)# switchport mode access<br>Switch(config-if-range)# switchport port-security<br>Switch(config-if-range)# switchport port-security maximum 1<br>Switch(config-if-range)# switchport port-security violation shutdown<br>Switch(config-if-range)# switchport port-security mac-address sticky</p>
<p>Detailed Command Explanation:<br>switchport port-security: Enables the security engine on the port.<br>maximum 1: Only allows one device to connect to this port.<br>violation shutdown: If a second MAC address is detected, the port is immediately disabled.<br>mac-address sticky: The switch “remembers” the first MAC address it sees and writes it into the running configuration.</p>
<p>B. Management Hardening (SSH): Disable Telnet and enable encrypted SSH access.</p>
<p>(config)# ip domain-name securelab.com<br>(config)# crypto key generate rsa<br>! Choose 1024<br>(config)# username admin privilege 15 secret cisco123<br>(config)# line vty 0 4<br>(config-line)# login local<br>(config-line)# transport input ssh</p>
<p>Detailed Command Explanation:<br>crypto key generate rsa: Generates the encryption keys needed for SSH.<br>line vty 0 4: Configures the virtual terminal lines (remote access).<br>transport input ssh: This is the critical security step—it explicitly forbids Telnet and only permits SSH.</p>
<p>6- Endpoint Addressing (Manual Configuration):<br>Device,IP Address,Subnet Mask,Default Gateway(RESPECTIVELY)<br>PC2 (HQ LAN),192.168.10.10,255.255.255.0,192.168.10.1<br>Server1 (DMZ),172.16.0.10,255.255.255.0,172.16.0.1<br>PC3 (Branch LAN),192.168.20.10,255.255.255.0,192.168.20.1</p>
<p>TESTING:</p>
<p>VPN: Ping from PC2 to PC3. Then run show crypto isakmp sa. Status should be QM_IDLE.<br>DMZ: Ping from Server1 to PC2. It should fail (Blocked by ACL).<br>Hardening: Unplug PC2 and plug in a different PC. The switch port should turn red (Shutdown).</p>

</body>
</html>
