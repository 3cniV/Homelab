---
description: >-
  This project focuses on attacking hardened environments, automating advanced
  adversarial methodologies, discovering modern web/API vulnerabilities, and
  analyzing defensive strategies.
---

# Project RedLine: An Aggressive Pentesting & Evasion Homelab

## Advanced Network Architecture, Recon Automation & Firewall Evasion

```
                                [ PUBLIC INTERNET ]
                                        │
                                    (WAN DHCP)
                                        ▼
                           ┌─────────────────────────┐
                               │ pfSense Firewall │
                           └────┬───────────────┬────┘
                                       │ │
                                 (em1) ▼ ▼ (em2)
                    ┌──────────────────┐ ┌──────────────────┐
                         │ DMZ Network │ │ Corporate LAN │
                     │ 192.168.10.0/24 │ │ 192.168.20.0/24 │
                    └────────┬─────────┘ └────────┬─────────┘
                                       │ │
                                       ▼ ▼
                       ┌───────────────┐ ┌───────────────┐
                       │ Ubuntu Server │ │Windows Server │
                       │ 192.168.10.12 │ │ 192.168.20.10 │
                       └───────────────┘ └───────────────┘
```

## Milestone 1: Local Privilege Escalation & Administrator Account Recovery.

During initial setup, access to the Windows Server 2022 template was blocked due to lost credential configurations. I implemented a local privilege escalation attack via binary swapping through the hypervisor recovery path.

#### Execution Methodology

1. **Boot Order Modification:** I modified the VirtualBox VM storage system to place the Windows Server installation ISO at position 1 by making the Optical drive the priority.

<figure><img src=".gitbook/assets/redline1.png" alt=""><figcaption></figcaption></figure>

2. **Recovery Shell Spawning:** I then rebooted the system and interrupted the installation sequence using the native shortcut `Shift + F10` to drop into a system-level command shell (`X:\\Sources>`).

<figure><img src=".gitbook/assets/redline2.png" alt=""><figcaption></figcaption></figure>

3. **The Utilman Exploit:** I modified the pre-login behavior of the windows server in order to validate a local privilege escalation path by creating a backup of the Utility Manager accessibility binary using:

```cmd
copy d:\\windows\\system32\\utilman.exe d:\\
copy d:\\windows\\system32\\cmd.exe d:\\windows\\system32\\utilman.exe
```

4. **Access Recovery**: After rebooting to the Windows sign-in screen, the test confirmed pre-authentication code execution with `NT AUTHORITY\\SYSTEM` privileges. I then restored access to the locked local profile and completed sign-in:

```
net user Administrator Password                                                                                 
```

## Milestone 2: Network Stack Migration & Core Layer Integration

Upon initial access, the Windows Server workstation pulled an IP allocation mapping directly inside the untrusted `192.168.10.0/24` DMZ block due to lingering deployment scripts.

#### Resolving Static Interface Conflicts

Because the host OS still used a static configuration, `ipconfig /release` initially failed with a permissions-related state error. Therefore, the following steps were taken to resolve it:

1. I opened the **Network Connections** with `ncpa.cpl`.
2. Then I changed the adapter's Internet Protocol Version 4 (TCP/IPv4) settings from static values to **Obtain an IP address automatically (DHCP)**.
3. Move the VirtualBox internal adapter from the `DMZ` switch to the `CorpLAN` internal network.
4. Verified the updated network state:
   * IPv4 Address: `192.168.20.10`
   * Default Gateway: `192.168.20.1`
   * Domain Suffix: `home.arpa`

### Milestone 3: Firewall Interface Alignment & Custom Branding

The default pfSense interface labels were updated to match the lab network design.

1. Temporarily disabled packet filtering from the pfSense console to prevent early lockout during reconfiguration:

```
pfctl -d
```

2. Opened the administrative interface at `https://192.168.20.1` in Microsoft Edge.
3. Navigated to **Interfaces → Assignments** and updated the interface names:
   * LAN (`em1`) $$ $\rightarrow$ $$ Renamed to `DMZ` (Gateway: `192.168.10.1`)
   * OPT1 (`em2`) $$ $\rightarrow$ $$ Renamed to `CorpLAN` (Gateway: `192.168.20.1`)



### Milestone 4: Outbound NAT & Granular Traffic Control

Due to optional network layers (like `OPT1`/`CORPLAN`) being treated as untrusted blocks by default, specialized rules had to be written to manage internet translation paths.

#### Firewall Traffic Matrix Configuration

A broad outbound permission rule was built inside Firewall -> Rules -> CorpLAN to establish global internet reachability while protecting internal boundaries:

<figure><img src=".gitbook/assets/redline3.png" alt=""><figcaption></figcaption></figure>

#### Outbound Network Address Translation (NAT)

Verified under Firewall -> NAT -> Outbound that the network translation routing system was generating automatic mappings to translate private space packets out through the external gateway:

* Target Network: `192.168.20.0/24` $$ $\rightarrow$ $$ Map to `WAN address`

<figure><img src=".gitbook/assets/redline4.png" alt=""><figcaption></figcaption></figure>

#### DNS Resolution Tuning

To fix VirtualBox loops, explicit external query routing targets were defined under System -> General Setup:

* Primary DNS Server: `8.8.8.8`
* Secondary DNS Server: `1.1.1.1`
* DNS Resolution Behavior: Switched to _“Use remote DNS Servers, ignore local DNS”_ to enforce definitive public resolution paths.

