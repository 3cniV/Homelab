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

1. **Boot Order Modification:** I modified the VirtualBox VM storage system to place the Windows Server installation ISO at position 1  by shifting the Optical drive to the top.
2. **Recovery Shell Spawning:** I then interrupted the standard installer setup wizard sequence using the native shortcut `Shift + F10` to drop into a system-level command shell (`X:\\Sources>`).
3.  **Volume Discovery:** Mapped the local volumes to identify the primary operating system footprint on the storage layout:

    ```cmd
    dir d:\\windows\\system32\\utilman.exe
    ```
