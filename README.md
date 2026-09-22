# Enterprise Network Lab

> Virtualized enterprise network lab designed and implemented using Hyper-V, pfSense, Windows Server, Active Directory, Ubuntu Server and Windows Client.

## 📌 Overview

This project is a virtualized enterprise network laboratory created to reproduce a small corporate network environment.

The lab focuses on:

* Network segmentation using VLANs
* Inter-VLAN routing
* Firewall and security policies
* NAT and Internet connectivity
* Active Directory and centralized authentication
* DNS and DHCP services
* DHCP Relay
* DMZ architecture
* Network troubleshooting
* Infrastructure documentation

The project was designed as a practical environment for studying and demonstrating enterprise networking concepts.

```

The Hyper-V virtual switch `LAB-LAN` carries the internal VLAN traffic.

pfSense provides:

* Inter-VLAN routing
* Firewalling
* NAT
* Internet access
* DHCP Relay
* Network segmentation

---

## 🌐 VLAN & IP Addressing

| VLAN    | Purpose             | Network         | Gateway      |
| ------- | ------------------- | --------------- | ------------ |
| VLAN 10 | Server / Management | `10.10.10.0/24` | `10.10.10.1` |
| VLAN 20 | Clients             | `10.10.20.0/24` | `10.10.20.1` |
| VLAN 30 | DMZ                 | `10.10.30.0/24` | `10.10.30.1` |

### Main IP addresses

| Device         | IP Address    | Role                 |
| -------------- | ------------- | -------------------- |
| pfSense        | `10.10.10.1`  | LAN / Native VLAN 10 |
| Hyper-V Host   | `10.10.10.3`  | Management           |
| DC01           | `10.10.10.10` | AD DS / DNS / DHCP   |
| pfSense VLAN20 | `10.10.20.1`  | Client Gateway       |
| pfSense VLAN30 | `10.10.30.1`  | DMZ Gateway          |
| Web01          | `10.10.30.10` | DMZ Server           |

---

## 🖥️ Virtual Machines

| VM       | OS                  | VLAN        | Main Role          |
| -------- | ------------------- | ----------- | ------------------ |
| pfSense  | pfSense             | WAN + VLANs | Firewall / Router  |
| DC01     | Windows Server 2025 | VLAN 10     | AD DS / DNS / DHCP |
| Client01 | Windows 10 Pro      | VLAN 20     | Domain Client      |
| Web01    | Ubuntu Server       | VLAN 30     | DMZ Server         |

---

## 🔐 Security Architecture

The network is segmented into separate security zones.

### VLAN 10 — Server / Management

Contains infrastructure services such as:

* Active Directory
* DNS
* DHCP
* Management interfaces

### VLAN 20 — Clients

Contains user workstations joined to the `corp.lab` domain.

### VLAN 30 — DMZ

Used to isolate externally exposed or less-trusted services from the internal network.

### Firewall Policy

The DMZ is intentionally restricted from accessing internal networks.

```text
VLAN30 → VLAN10     ❌ BLOCK
VLAN30 → VLAN20     ❌ BLOCK
VLAN30 → INTERNET   ✅ ALLOW

VLAN20 → VLAN10     ✅ ALLOW
VLAN20 → VLAN30     ✅ ALLOW
VLAN20 → INTERNET   ✅ ALLOW
```

Firewall rules are evaluated from top to bottom.

---

## 🏢 Active Directory

The Windows Server `DC01` was configured as the first domain controller for:

```text
corp.lab
```

Services provided by DC01:

* Active Directory Domain Services
* DNS
* DHCP

The Windows client `Client01` was joined to the domain and receives its network configuration through DHCP.

---

## 🌐 DHCP & DNS

### DHCP

DHCP is provided by Windows Server.

Client subnet:

```text
10.10.20.0/24
```

DHCP range:

```text
10.10.20.100 - 10.10.20.200
```

The DHCP server provides:

* IP address
* Default gateway
* DNS server
* Domain name

Because the DHCP server is located in VLAN 10 while clients are located in VLAN 20, pfSense performs DHCP Relay.

```text
Client01
   │
   │ DHCP
   ▼
pfSense VLAN20
   │
   │ DHCP Relay
   ▼
DC01
10.10.10.10
```

---

## 🔥 Firewall & DMZ Testing

Firewall policies were tested using real connectivity tests and packet captures.

Example:

```text
Web01 (10.10.30.10)
        │
        ├──→ DC01 (10.10.10.10)   BLOCKED
        │
        ├──→ Client01 (10.10.20.x) BLOCKED
        │
        └──→ Internet              ALLOWED
```

The firewall configuration was also validated using:

* Firewall logs
* Packet Capture
* State table
* ICMP tests
* Connectivity tests

---

## 🧪 Connectivity Testing

| Source   | Destination    | Expected | Result |
| -------- | -------------- | -------: | -----: |
| Client01 | DC01           |    ALLOW |      ✅ |
| Client01 | Web01          |    ALLOW |      ✅ |
| Client01 | Internet       |    ALLOW |      ✅ |
| Web01    | DC01           |    BLOCK |      ✅ |
| Web01    | Client01       |    BLOCK |      ✅ |
| Web01    | Internet       |    ALLOW |      ✅ |
| Client01 | `corp.lab` DNS |    ALLOW |      ✅ |

---

## 🛠️ Troubleshooting

One of the main troubleshooting scenarios involved a DMZ firewall rule that initially did not block traffic as expected.

### Problem

`Web01` in VLAN 30 was able to reach `DC01` in VLAN 10 despite a configured firewall rule.

### Investigation

The following were checked:

* Firewall rule order
* Firewall logs
* Packet capture
* State table
* Floating rules
* VLAN configuration
* Destination subnet

### Root Cause

VLAN 10 was configured as the native/untagged VLAN on the Hyper-V trunk.

The automatic subnet reference for the tagged VLAN 10 interface did not correspond to the actual `10.10.10.0/24` network.

### Solution

The firewall rule was changed to explicitly use:

```text
Destination: 10.10.10.0/24
```

After the change:

```text
Web01 → DC01       BLOCKED
Web01 → Client01   BLOCKED
Web01 → Internet   ALLOWED
```

This troubleshooting process helped validate both the firewall configuration and the underlying VLAN architecture.

---

## Troubleshooting Commands

### Linux

```bash
ip addr
ip route
ping <IP>
traceroute <IP>
ss -lntp
systemctl status <service>
nslookup <name>
dig <name>
curl <URL>
```

### Windows

```powershell
ipconfig /all
ping <IP>
tracert <IP>
nslookup <name>
Test-NetConnection <IP> -Port <port>
route print
```

### Active Directory / DNS

```powershell
Get-ADDomain
nslookup corp.lab
nslookup -type=SRV _ldap._tcp.dc._msdcs.corp.lab
```

### Hyper-V

```powershell
Get-VM
Get-VMNetworkAdapter
Get-VMNetworkAdapterVlan
---

## Technologies

* Hyper-V
* pfSense
* Windows Server 2025
* Windows 10 Pro
* Ubuntu Server
* Active Directory
* DNS
* DHCP
* 802.1Q VLANs
* NAT
* Firewall
* Wireshark
* PowerShell

---

## Disclaimer

This is a personal educational laboratory created for learning and experimentation.

No production credentials, private keys, passwords or sensitive infrastructure information are included in this repository.
