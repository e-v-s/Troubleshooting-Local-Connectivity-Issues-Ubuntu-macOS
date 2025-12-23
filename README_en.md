dsadsa

# Troubleshooting: Local Connectivity Issues (Ubuntu ↔ macOS)

## Menu

- [Troubleshooting: Local Connectivity Issues (Ubuntu ↔ macOS)](#troubleshooting-local-connectivity-issues-ubuntu--macos)
  - [Menu](#menu)
  - [Versions](#versions)
  - [Technologies Used](#technologies-used)
  - [Skills Used](#skills-used)
  - [Context](#context)
  - [Observed Symptoms](#observed-symptoms)
  - [Step 1: IPv4 vs IPv6 Diagnosis](#step-1-ipv4-vs-ipv6-diagnosis)
    - [Observation](#observation)
    - [Conclusion](#conclusion)
    - [Fix](#fix)
  - [Step 2: Firewall Verification](#step-2-firewall-verification)
    - [Ubuntu](#ubuntu)
    - [macOS](#macos)
  - [Step 3: SSH Server Verification on Ubuntu](#step-3-ssh-server-verification-on-ubuntu)
    - [Symptoms](#symptoms)
    - [Conclusion](#conclusion-1)
    - [Fix](#fix-1)
  - [Step 4: libvirt / iptables Interference](#step-4-libvirt--iptables-interference)
    - [Test Performed](#test-performed)
    - [Conclusion](#conclusion-2)
  - [Step 5: Cross-Network Testing](#step-5-cross-network-testing)
  - [Root Cause](#root-cause)
    - [Observed Behavior](#observed-behavior)
    - [Direct Impact](#direct-impact)
    - [Evidence](#evidence)
  - [Practical Solutions Adopted](#practical-solutions-adopted)
  - [Lessons Learned](#lessons-learned)
  - [Conclusion](#conclusion-3)

## Versions

- [Portuguese Version](./README.md)
- **English (current)**

## Technologies Used

![Ubuntu](https://img.shields.io/badge/Ubuntu-E95420?logo=ubuntu&logoColor=white)
![macOS](https://img.shields.io/badge/macOS-000000?logo=apple&logoColor=white)
![KVM](https://img.shields.io/badge/KVM-00599C?logo=linux&logoColor=white)
![libvirt](https://img.shields.io/badge/libvirt-5A2D82)
![OpenSSH](https://img.shields.io/badge/OpenSSH-000000?logo=openssh&logoColor=white)
![iptables](https://img.shields.io/badge/iptables-777777)
![Network Troubleshooting](https://img.shields.io/badge/Networking-Debugging-blue)

## Skills Used

- Network troubleshooting (L2/L3/L7)
- IPv4 vs IPv6 diagnostics
- Local firewall analysis (ufw / iptables)
- Linux service debugging (systemctl, ssh, iptables, ufw)
- ISP router behavior analysis
- Cross-network connectivity testing
- Structured troubleshooting methodology

## Context

While setting up my homelab (Ubuntu as the host with KVM/libvirt and macOS as the workstation), I ran into several issues when trying to establish local communication between an Ubuntu laptop (Lenovo ThinkPad T480) and a MacBook on the same residential Wi-Fi network.

The initial symptoms pointed to failures in seemingly simple services (local HTTP, SSH, SCP), which led to a full troubleshooting process covering operating system configuration, firewall rules, virtualization layers, and network behavior.

This document describes the real troubleshooting process, the tests performed, the false positives discarded, and the final root cause.

## Observed Symptoms

- Internet access worked normally on both devices
- `ping www.google.com` worked
- `ping 8.8.8.8` initially failed (IPv4)
- `python3 -m http.server` started correctly on Ubuntu
- The port appeared in `LISTEN` state (`ss -lntp`)
- Accessing the Ubuntu host from macOS via browser failed
- `curl` returned `recv failure: connection reset by peer`
- `nc` (netcat) was able to connect
- SSH (`ssh user@ip`) did not connect
- SCP initially did not work

## Step 1: IPv4 vs IPv6 Diagnosis

### Observation

- `ping www.google.com` worked
- `ping 8.8.8.8` did not work
- `ip route` showed only the `virbr0` network

### Conclusion

Ubuntu was operating with IPv6 only on the Wi-Fi interface, which explained:

- Internet access working normally
- Failures in local IPv4-based services

### Fix

- Forced IPv4 via NetworkManager
- Reconnected to the Wi-Fi network
- Confirmed the presence of a default IPv4 route using `ip route`

After this fix, `ping 8.8.8.8` started working.

**The issue persisted for HTTP/SSH.**

## Step 2: Firewall Verification

### Ubuntu

- `ufw status` → inactive
- No explicit rules blocking ports

### macOS

- Firewall disabled
- No proxy configured
- No active VPN

Firewall was ruled out as the cause.

## Step 3: SSH Server Verification on Ubuntu

### Symptoms

- `ssh localhost` initially returned `connection refused`
- `ss -lntp | grep :22` showed no listening port
- `journalctl -xeu ssh` showed no errors

### Conclusion

The `openssh-server` service was not running correctly.

### Fix

- Reinstalled `openssh-server`
- Manually started the `ssh` service via `systemctl`
- Confirmed `LISTEN` state on port 22

After that, `ssh -v localhost` worked correctly.

## Step 4: libvirt / iptables Interference

Even with `ufw` disabled, `libvirt` had automatically inserted rules into `iptables`.

### Test Performed

- Flushed all `iptables` rules
- Set default policies to `ACCEPT`
- `ssh localhost` continued to work
- Remote SSH still did not work

### Conclusion

The block was not happening at the Ubuntu `iptables` level.

## Step 5: Cross-Network Testing

Connected both Ubuntu and macOS to a mobile hotspot:

- SSH worked immediately
- Local HTTP worked

## Root Cause

The issue was not related to the operating systems, but to the residential Wi-Fi router **F6600R (ZTE, provided by the ISP)**.

This router silently enforces client isolation between Wi-Fi devices, even on the main SSID.

### Observed Behavior

- Wi-Fi ↔ Wi-Fi traffic blocked
- Wi-Fi ↔ Gateway allowed
- Internet access works normally
- Lateral communication between devices is not allowed
- No visible option in the web interface to disable this behavior

### Direct Impact

- SSH blocked
- Local HTTP blocked
- Only short-lived connections (e.g., `nc`) allowed
- No explicit errors at the OS level

### Evidence

- Communication fails on the ZTE Wi-Fi network
- Communication works immediately via mobile hotspot
- No additional configuration changes required

## Practical Solutions Adopted

- Use mobile hotspot when local transfers are needed
- Use SSH/SCP only on trusted networks
- Avoid relying on ISP-provided Wi-Fi for lab environments
- Plan to use a personal router for the homelab

## Lessons Learned

- Internet connectivity does not imply local connectivity
- IPv6 can mask IPv4-related issues
- Not all blocks are visible at the firewall or OS level
- ISP-provided routers are not reliable environments for technical labs
- Cross-network testing is essential to identify the real root cause

## Conclusion

After a full investigation covering operating system configuration, firewall rules, virtualization layers, network behavior, and cross-network testing, the issue was identified as Wi-Fi client isolation enforced by the ZTE router.

No misconfiguration was found on either Ubuntu or macOS.

This troubleshooting reinforces the importance of validating the physical and logical network layers before assuming software-level failures.
