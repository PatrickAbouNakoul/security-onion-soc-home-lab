# Security Onion Lab Troubleshooting

## Overview

A significant part of this Security Onion home lab involved troubleshooting the environment before I achieved reliable end-to-end monitoring.

The main challenge was not simply installing Security Onion. It was making sure the attacker, victim, and monitoring interfaces were connected in a way that gave Security Onion visibility into the traffic I wanted to analyse.

This document records the main problems I encountered, how I investigated them, and what I learned from resolving them.

## Problem 1: Testing the Management Interface

My first traffic-generation attempt involved scanning the Security Onion management IP directly from the Windows host.

```bash
nmap -A 192.168.56.10
```

### Result

The scan did not generate the detections I expected.

### Root Cause

The Security Onion management interface was being used for administration, but it was not the network path I ultimately needed to monitor.

The platform itself was functioning, but the lab architecture was not suitable for the traffic-generation workflow I wanted.

### Resolution

I redesigned the lab around three distinct roles:

```text
Kali Linux
Attacker
10.10.10.20
      |
      v
Ubuntu
Victim
10.10.10.30
      |
      v
Traffic monitored by Security Onion
```

This allowed Security Onion to observe attacker-to-victim traffic on the monitored network instead of relying on traffic sent directly to its management interface.

## Problem 2: Monitoring Interface and Traffic Visibility

During the earlier deployment, Security Onion was not consistently seeing the traffic I expected.

I investigated:

```bash
sudo so-status
ip -br a
ip route
```

I also checked whether:

- the correct monitoring interface had been selected
- `bond0` existed
- the monitoring interface was active
- the attacker and victim were on the intended network
- traffic was actually travelling across the monitored segment

### Lesson

A healthy IDS or monitoring platform does not automatically mean the correct network traffic is visible to it.

Monitoring depends heavily on network architecture and interface placement.

## Problem 3: Security Onion Deployment Issues

The first Security Onion deployment became increasingly difficult to troubleshoot.

Problems included:

- Alerts page errors
- `so-test` being unable to complete because the VM did not have internet access
- `so-monitor-add` failing in the earlier deployment
- inconsistent monitoring-interface selection
- expected traffic not appearing
- VirtualBox networking behaving differently from what I initially assumed

Rather than continuing to modify a partially broken deployment, I decided to rebuild Security Onion from scratch.

## Rebuilding Security Onion

The rebuilt deployment used a cleaner configuration.

### Hostname

```text
sohome
```

### Management Interface

```text
Interface: enp0s3
IP Address: 192.168.56.10/24
Gateway: 192.168.56.1
```

### Monitoring Interface

```text
Interface: enp0s8
Monitoring Interface: bond0
```

### Deployment Type

```text
EVAL
```

I also configured access for the management subnet:

```text
192.168.56.0/24
```

## Problem 4: Standard Mode Required Internet Access

During the rebuilt installation, I initially ran Security Onion setup in Standard mode.

The setup attempted to reach external Security Onion resources.

Because the VM did not have internet access, the installation generated errors while trying to resolve or contact external hosts.

### Resolution

I stopped the Standard mode setup and relaunched configuration using:

```bash
sudo SecurityOnion/setup/so-setup iso
```

I then used **Airgap mode**, which was better suited to the isolated lab.

Security Onion subsequently completed setup successfully.

## Deployment Verification

After the rebuild completed, I verified the platform using:

```bash
sudo so-status
ip -br a
```

I confirmed:

- core Security Onion services were running
- `enp0s3` had `192.168.56.10/24`
- `bond0` existed and was active
- the platform was ready for monitoring

This was the first point where I was confident the Security Onion deployment itself was stable.

## Problem 5: Ubuntu Addressing

The Ubuntu victim also required some troubleshooting.

I initially experimented with temporary addressing, but the configuration was not being retained as cleanly as I wanted.

I inspected the network configuration using:

```bash
ip -br a
ip route
nmcli device status
nmcli connection show
```

I then configured a static IPv4 address using NetworkManager:

```bash
sudo nmcli connection mod "Wired connection 1" ipv4.method manual ipv4.addresses 10.10.10.30/24 ipv6.method ignore

sudo nmcli connection down "Wired connection 1"

sudo nmcli connection up "Wired connection 1"
```

I verified the result using:

```bash
ip -br a
ip addr show dev enp0s3
ip route
```

The victim was then stable at:

```text
10.10.10.30/24
```

## Problem 6: Kali Network Connectivity

Kali also experienced connectivity problems while I was finalising the monitored segment.

At one point the system returned:

```text
Network is unreachable
```

I checked and corrected the interface addressing and routing.

```bash
ip -br a
sudo ip addr add 10.10.10.20/24 dev eth0
sudo ip link set eth0 up
ip route
```

The final Kali configuration was:

```text
Interface: eth0
IP Address: 10.10.10.20/24
Route: 10.10.10.0/24 dev eth0
```

## End-to-End Validation

Before investigating IDS alerts, I confirmed that the underlying traffic flow worked.

### ICMP

```bash
ping -c 4 10.10.10.30
```

### HTTP

```bash
curl http://10.10.10.30:8000/
```

Once both tests succeeded, I knew:

- Kali could communicate with Ubuntu
- the HTTP service was reachable
- the systems were on the correct monitored segment
- the environment was ready for detection testing

## Troubleshooting Methodology

The general troubleshooting process I used was:

```text
Check Platform Health
        ↓
Check Interfaces
        ↓
Check IP Addressing
        ↓
Check Routing
        ↓
Check Monitoring Path
        ↓
Check Service Availability
        ↓
Generate Simple Test Traffic
        ↓
Validate Detection
```

## Key Lessons

### Network visibility comes first

A monitoring platform cannot analyse traffic it cannot see.

### Verify the fundamentals before blaming the IDS

Before assuming Security Onion or Suricata is failing, verify:

```text
Interfaces
IP addressing
Routes
Link state
Traffic path
Service availability
```

### A clean rebuild can be more effective than endless patching

Once the original configuration became unnecessarily complicated, rebuilding Security Onion with a cleaner design produced a much more reliable environment.

### Test progressively

I found it more effective to validate the environment in stages:

```text
Platform health
→ Network connectivity
→ Service connectivity
→ Basic traffic
→ Reconnaissance traffic
→ Alert validation
```

## Final Result

After correcting the architecture and rebuilding the environment, the lab successfully supported:

- Kali-to-Ubuntu communication
- HTTP service testing
- ICMP traffic
- Nmap reconnaissance
- Suricata alert generation
- Security Onion alert investigation

The troubleshooting process was one of the most valuable parts of the project because it strengthened my understanding of the relationship between network architecture, telemetry, and security monitoring.
