# Wi-Fi Evil Twin — Raspberry Pi Security Lab

![Platform](https://img.shields.io/badge/platform-Raspberry%20Pi%203-red)
![OS](https://img.shields.io/badge/OS-Raspberry%20Pi%20OS%2064--bit-blue)
![Status](https://img.shields.io/badge/status-complete-brightgreen)
![License](https://img.shields.io/badge/license-Educational%20Use%20Only-orange)

Built on a Raspberry Pi 3 B, this educational project recreates a complete Wi-Fi Evil Twin attack using only the built-in wireless chip. A rogue access point lures test devices, a custom DNS server forces a captive portal login page, and captured credentials appear live on a Flask monitoring dashboard. The project documents the full attack chain alongside defensive countermeasures, serving as a hands-on introduction to wireless security testing in a controlled lab environment.

---

## ⚠️ Disclaimer

**Educational use only.** This project was built and tested exclusively on hardware and networks I own. Unauthorized use against networks or devices you do not own is illegal.

---

## 🖥️ Hardware & Software

- **Device:** Raspberry Pi 3 Model B
- **OS:** Raspberry Pi OS 64-bit (Bookworm)
- **Wi-Fi:** Built-in Broadcom BCM43438 (wlan0)
- **Key Packages:** hostapd, dnsmasq, Python 3, Flask

---

## 🚀 Quick Start

```bash
# Clone or copy files to /root/

# Start the Evil Twin
sudo /root/start-evil-twin.sh

# Access the monitoring dashboard
# http://192.168.222.30:8080

# Connect a test device to "TestWiFi_Free"
# Browse to any URL — captive portal will appear
# Enter test credentials — they appear on the dashboard

# Stop the Evil Twin
sudo /root/stop-evil-twin.sh

/root/
├── start-evil-twin.sh          # Startup script
├── stop-evil-twin.sh           # Shutdown script
├── captive-portal.py           # Captive portal (port 80)
├── evil-monitor/
│   └── app.py                  # Flask dashboard (port 8080)
└── README.md                   # This file

/etc/
├── hostapd/hostapd.conf        # AP configuration
└── dnsmasq.conf                # DHCP + DNS configuration

Interface	IP	Role
eth0    	192.168.222.30    	Management & Internet
wlan0	    192.168.0.1	         Evil Twin AP Gateway
