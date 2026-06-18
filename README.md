---
description: >-
  This project focuses on attacking hardened environments, automating advanced
  adversarial methodologies, discovering modern web/API vulnerabilities, and
  analyzing defensive strategies.
---

# Project RedLine: An Aggressive Pentesting & Evasion Homelab

## Advanced Network Architecture, Recon Automation & Firewall Evasion

### Executive Summary

This project designs, builds, and stress-tests a three-tier zero-trust lab inside a virtualized sandbox. pfSense Community Edition acts as the perimeter firewall and routing layer by separating traffic across three zones: a WAN edge, a public-facing DMZ, and a restricted internal Corporate LAN (`CORPLAN`).

The goal is to replace a flat lab network with strict segmentation and controlled trust boundaries. The implementation includes outbound NAT, custom DNS behavior, and stateful firewall rules that govern inter-zone traffic.

The project also includes hands-on recovery and remediation work:

* **Infrastructure account recovery:** Used a boot-level binary swap (`Utilman.exe` hijack) to regain access to a locked administrative image.
* **Operating system stabilization:** Resolved Windows Server 2022 evaluation limits that caused forced shutdowns.
* **Security leak remediation:** Identified and fixed a cross-zone firewall rule leak that allowed lateral movement between the DMZ and the corporate management network.

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

1. **Boot Order Modification:** The VirtualBox VM storage system was modified to place the Windows Server installation ISO at position 1 by making the Optical drive the priority.

<figure><img src=".gitbook/assets/redline1.png" alt=""><figcaption></figcaption></figure>

2. **Recovery Shell Spawning:** The system was rebooted and the installation sequence was interrupted using the native shortcut `Shift + F10` to drop into a system-level command shell (`X:\\Sources>`).

<figure><img src=".gitbook/assets/redline2.png" alt=""><figcaption></figcaption></figure>

3. **The Utilman Exploit:** The pre-login behavior of the windows server was modified in order to validate a local privilege escalation path by creating a backup of the Utility Manager accessibility binary using the `utilman.exe` hijack:

```cmd
copy d:\\windows\\system32\\utilman.exe d:\\
copy d:\\windows\\system32\\cmd.exe d:\\windows\\system32\\utilman.exe
```

4. **Access Recovery**: After rebooting to the Windows sign-in screen, the test confirmed pre-authentication code execution with `NT AUTHORITY\\SYSTEM` privileges. Access was then restored to the locked local profile for sign-in:

```
net user Administrator Password                                                                                 
```

## Milestone 2: Network Stack Migration & Core Layer Integration

Upon initial access, the Windows Server workstation pulled an IP allocation mapping directly inside the untrusted `192.168.10.0/24` DMZ block due to lingering deployment scripts.

#### Resolving Static Interface Conflicts

Because the host OS still used a static configuration, `ipconfig /release` initially failed with a permissions-related state error. Therefore, the following steps were taken to resolve it:

1. **Network Connections** was opened with `ncpa.cpl`.
2. The adapter's Internet Protocol Version 4 (TCP/IPv4) settings were changed from static values to **Obtain an IP address automatically (DHCP)**.
3. The VirtualBox internal adapter was moved from the `DMZ` switch to the `CorpLAN` internal network.
4. The updated network state was verified:
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

### Milestone 5: Windows Evaluation Engine Stabilization

After external DNS routing was confirmed, the Windows Server 2022 instance began forcing automatic shutdowns during normal WAN-connected operation.

The issue was traced to evaluation license enforcement within the base image. To stabilize the management host, the Microsoft Software License Manager was run from an elevated Command Prompt to reinstall the license files and reset the trial state:

```
slmgr /rilc :: Re-installs evaluation license certificates to clear active volume corruption
slmgr /rearm :: Resets the 180-day evaluation trial validation timer back to baseline
shutdown /r /t 0
```

### Milestone 6: DMZ Re-provisioning & Netplan Remediation

With the Corporate LAN management tier stabilized, focus shifted to provisioning the public-facing application tier inside the isolated DMZ. After the original Ubuntu instance froze during credential maintenance, a clean Ubuntu Server deployment was used to restore a known-good state.

#### 1. Hypervisor Layer Realignment

Before boot, the VirtualBox network profile was reviewed. Adapter 1 was detached from stale interface groups and reassigned to an **Internal Network** named `DMZ`. This ensured the VM mapped directly to the pfSense `em1` interface.

#### 2. Netplan Layout and YAML Error Correction

During manual interface setup, Netplan returned an `inconsistent indentation` error caused by mixed spacing. The configuration in `/etc/netplan/00-installer-config.yaml` was rewritten with clean spacing to support DHCP-based addressing:

```
# This is the network config written by 'subiquity'

network:

  ethernets:

    enp0s3:

      match:

        macaddress: **:**:**:**:**:**

      set-name: enp0s3

      dhcp4: true

  version: 2
```

#### 3. DHCP Pool Activation

Inside the pfSense interface (Services -> DHCP Server -> DMZ), the local DHCP pool daemon was toggled to active status, mapping an available lease boundary spanning `192.168.10.10` to `192.168.10.50`. Running `sudo netplan apply` in the ubuntu server successfully initiated the handshake sequence, resulting in the Ubuntu asset catching a valid dynamic allocation of `192.168.10.10`.

<figure><img src=".gitbook/assets/redline6 (1).png" alt=""><figcaption></figcaption></figure>

<figure><img src=".gitbook/assets/redline6.1.png" alt=""><figcaption></figcaption></figure>

### Milestone 7: Zero-Trust Security Validation

The final milestone verified cross-zone isolation and confirmed that the firewall enforced the intended trust boundaries.

#### Test 1: Outbound WAN connectivity

* Action: Ran a basic network connectivity test from the Ubuntu Server host.

```
ping -c 4 1.1.1.1
```

* Result: Success. Confirms outbound NAT translated DMZ traffic through the WAN interface and allowed external connectivity for updates.

<figure><img src=".gitbook/assets/redline7.png" alt=""><figcaption></figcaption></figure>

#### Test 2: Lateral movement block

* Action: Attempted to reach the internal Corporate Management host from the public-facing Ubuntu Server.

```
ping -c 4 192.168.20.10
```

* Initial observation: Traffic crossed segments due to an overly broad default pass rule left behind during interface renaming. This exposed a path for lateral movement from the DMZ into the corporate network.
* Remediation: Added an explicit deny rule at the top of **Firewall → Rules → DMZ** to block cross-subnet traffic before routing occurred.

<figure><img src=".gitbook/assets/redline8.png" alt=""><figcaption></figcaption></figure>

* Resul&#x74;**:** Re-running the ping test returned 100% packet loss. This confirmed full boundary isolation between the DMZ and the corporate network.

<figure><img src=".gitbook/assets/redline9.png" alt=""><figcaption></figcaption></figure>

### Conclusion

This project successfully delivered a segmented three-tier lab that mirrors core zero-trust design principles in a controlled environment. The final architecture enforced clear separation between the WAN edge, the public-facing DMZ, and the internal corporate network while preserving the services required for administration, DNS resolution, DHCP, and outbound connectivity.

The project also moved beyond simple deployment. It required account recovery, operating system stabilization, network re-provisioning, firewall rule correction, and repeated validation under real test conditions. Each issue exposed a practical failure point, and each fix strengthened the environment.

The final validation confirmed two critical outcomes. Public-facing systems in the DMZ retained outbound internet access through NAT, and lateral movement into the corporate network was fully blocked by explicit firewall policy. Together, these results demonstrate that the lab is not only functional, but defensible.

This environment now serves as a stable foundation for advanced offensive testing, firewall evasion research, and future adversary simulation workflows.
