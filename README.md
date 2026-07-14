# Pi-hole Setup Guide

**Ubuntu Server · Raspberry Pi · Netplan**

A practical setup guide for installing **Pi-hole** on a Raspberry Pi running **Ubuntu Server**, with a focus on configuring **Wi‑Fi networking** and assigning a **static IP address using Netplan**.

This guide is intended to get you operational quickly, while also serving as a reusable Netplan YAML reference for future setups.

> **Note**: This guide primarily targets Raspberry Pi devices using **Wi‑Fi**. If your Pi has **only an Ethernet connection** (or no wireless hardware at all), skip the Wi‑Fi sections and follow [Ethernet‑Only Setup (No Wi‑Fi)](#ethernet-only-setup-no-wi-fi) instead — everything else in the guide works the same.

---

## Features

* Wi‑Fi troubleshooting for Ubuntu Server on Raspberry Pi
* Netplan configuration template for Wi‑Fi
* Netplan configuration template for Ethernet‑only devices
* Static IP address assignment
* Custom DNS configuration (Cloudflare)
* Pi‑hole installation and initial setup

---

## How It Works

1. Verify that the network interface is available (wireless or wired)
2. Configure Wi‑Fi using Netplan — or skip this for Ethernet‑only devices
3. Apply and validate Netplan changes
4. Assign a static IP address
5. Install Pi‑hole
6. Secure the Pi‑hole admin interface

---

## Prerequisites

* Raspberry Pi with Ubuntu Server installed
* Active Wi‑Fi network (SSID and password) **or** an Ethernet cable connected to your router/switch
* Sudo privileges
* Internet connectivity

---

## Wi‑Fi Setup (If Interface Is Down)

If Wi‑Fi does not work out of the box, follow this section. Otherwise, you may skip ahead.

### Check Wireless Interface

```bash
ip a
```

Look for a wireless interface such as `wlan0`. This is the interface we will configure.

### Edit Netplan Configuration

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Example Wi‑Fi Netplan structure:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      optional: true
      dhcp4: true
      dhcp6: true
  wifis:
    wlan0:
      optional: true
      dhcp4: yes
      regulatory-domain: "NA"
      access-points:
        "YourSSID":
          password: "YourPassword"
```

Replace:

* `YourSSID` with your Wi‑Fi network name
* `YourPassword` with your Wi‑Fi password

### Apply Netplan Changes

```bash
sudo netplan generate
sudo netplan apply
```

---

## Static IP Assignment

A static IP is recommended for Pi‑hole so clients can reliably use it as a DNS server.

### Edit Netplan Configuration

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Example static IP configuration:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      optional: true
      dhcp4: true
      dhcp6: true
  wifis:
    wlan0:
      optional: true
      dhcp4: no
      addresses:
        - 10.0.0.200/24
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.1.1.3
      access-points:
        "YourSSID":
          password: "YourPassword"
      routes:
        - to: 0.0.0.0/0
          via: 10.0.0.254
      regulatory-domain: "NA"
```

> Remember to replace `YourSSID` and `YourPassword` with your own Wi‑Fi credentials.

---

## Configuration Breakdown

### Static IP

```yaml
addresses:
  - 10.0.0.200/24
```

* Assigns a static IPv4 address to the Raspberry Pi
* `/24` corresponds to subnet mask `255.255.255.0`

### DNS Configuration

```yaml
nameservers:
  addresses:
    - 1.1.1.1
    - 1.1.1.3
```

* **1.1.1.1** – Cloudflare primary DNS
* **1.1.1.3** – Cloudflare malware and adult‑content blocking DNS

**Reasoning**:

* Avoid ISP DNS hijacking
* Maintain consistent performance
* Improve privacy and security

---

## Apply and Verify

Validate configuration:

```bash
sudo netplan generate
```

Apply configuration:

```bash
sudo netplan apply
```

Verify IP assignment:

```bash
ip a
```

Your IP address should match the static address you configured.

---

## Ethernet-Only Setup (No Wi-Fi)

Some Raspberry Pi models have no wireless hardware (e.g. early Pi 1/2 boards), and in many setups a wired connection is simply the better choice — it is faster, more stable, and ideal for an always‑on DNS server like Pi‑hole. If your Pi is connected **only via an Ethernet cable**, this section replaces the Wi‑Fi sections above. Everything after this section (Pi‑hole installation and setup) is identical.

### Check the Ethernet Interface

```bash
ip a
```

Look for a wired interface such as `eth0`. If the cable is connected, the interface should show `state UP` and (with DHCP) already have an IP address.

If the interface is present but shows `state DOWN`, check that:

* The Ethernet cable is firmly seated on both ends
* The router/switch port is active (link LEDs on the Pi's Ethernet port should light up)

### Edit Netplan Configuration

```bash
sudo nano /etc/netplan/50-cloud-init.yaml
```

Example **Ethernet‑only static IP** configuration — note there is no `wifis:` block at all:

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: no
      addresses:
        - 10.0.0.200/24
      nameservers:
        addresses:
          - 1.1.1.1
          - 1.1.1.3
      routes:
        - to: 0.0.0.0/0
          via: 10.0.0.254
```

Adjust for your network:

* `10.0.0.200/24` — the static IP you want the Pi to have (must be inside your router's subnet and outside its DHCP pool if possible)
* `10.0.0.254` — your router's (gateway) IP address; find it with `ip route | grep default`
* The nameservers are the same Cloudflare DNS servers explained in the [Configuration Breakdown](#configuration-breakdown) above

> **Tip**: Since there are no Wi‑Fi credentials involved, this is the simplest possible Netplan configuration — no SSID, password, or regulatory domain needed.

### Apply and Verify

```bash
sudo netplan generate
sudo netplan apply
```

> **Warning**: If you are connected over SSH and change the IP address, your session will drop when the new address takes effect. Reconnect using the new static IP: `ssh <user>@10.0.0.200`

Verify the address and connectivity:

```bash
ip a
ping -c 3 1.1.1.1
```

Your `eth0` interface should show the static address you configured, and the ping confirms internet connectivity. You can now continue with the [Pi‑hole Installation](#pihole-installation) section below.

---

## Pi‑hole Installation

Once the static IP is configured, proceed with Pi‑hole installation.

### Install Pi‑hole

```bash
curl -sSL https://install.pi-hole.net | bash
```

Follow the interactive installer and accept the recommended defaults.

### Set Pi‑hole Admin Password

After installation:

```bash
pihole setpassword
```

This secures access to the Pi‑hole web admin portal.

---

## Useful Documentation

* Ubuntu Server documentation: [https://ubuntu.com/server/docs](https://ubuntu.com/server/docs)
* Pi‑hole documentation: [https://docs.pi-hole.net/](https://docs.pi-hole.net/)
* Pi‑hole install guide: [https://docs.pi-hole.net/main/basic-install/](https://docs.pi-hole.net/main/basic-install/)
* Pi‑hole donation page: [https://pi-hole.net/donate/](https://pi-hole.net/donate/)

---

## Notes

* This guide provides a minimal but functional Pi‑hole setup
* Intended as a baseline configuration
* Documentation will be updated as tools and workflows change

---

## License

MIT License
