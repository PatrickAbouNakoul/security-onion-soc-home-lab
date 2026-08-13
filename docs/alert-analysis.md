# Suricata Alert Analysis

## Investigation Overview

As part of the Security Onion SOC home lab, I generated controlled reconnaissance traffic from a Kali Linux attacker machine toward an Ubuntu victim system.

Security Onion successfully captured the activity and generated multiple Suricata alerts.

One of the clearest alerts investigated was:

```text
ET SCAN Nmap Scripting Engine User-Agent Detected
```

The purpose of this investigation was to correlate the alert with the activity generated from Kali and determine what the network evidence represented.

## Environment

| System | Role | IP Address |
| --- | --- | --- |
| Kali Linux | Attacker | `10.10.10.20` |
| Ubuntu | Victim | `10.10.10.30` |
| Security Onion | Monitoring | `192.168.56.10` |

The Ubuntu victim hosted a simple Python HTTP server on TCP port `8000`.

```bash
python3 -m http.server 8000
```

## Activity Generated

From Kali Linux, I performed service and reconnaissance testing against the Ubuntu victim.

```bash
nmap -sV -p 8000 10.10.10.30
```

I also performed a more comprehensive Nmap scan:

```bash
nmap -A 10.10.10.30
```

These scans generated reconnaissance traffic that was visible to Security Onion.

## Alerts Observed

Suricata generated several alerts associated with the controlled activity, including:

```text
ET INFO Python SimpleHTTP ServerBanner

ET SCAN Nmap Scripting Engine User-Agent Detected

ET SCAN Possible Nmap User-Agent Observed

GPL ICMP PING *NIX

ET HUNTING curl User-Agent to Dotted Quad

ET SCAN NMAP OS Detection Probe
```

Multiple additional scan-related alerts were also generated during the Nmap activity.

## Selected Alert

The primary alert investigated was:

```text
ET SCAN Nmap Scripting Engine User-Agent Detected
```

## Alert Evidence

Important fields associated with the event included:

| Field | Value |
| --- | --- |
| Source IP | `10.10.10.20` |
| Destination IP | `10.10.10.30` |
| Destination Port | `8000` |
| Monitoring Interface | `bond0` |
| Application Protocol | HTTP |
| HTTP Method | `OPTIONS` |
| HTTP Request | `OPTIONS / HTTP/1.1` |
| User-Agent | `Nmap Scripting Engine` |

## Evidence Correlation

The source IP `10.10.10.20` belonged to the Kali attacker machine.

The destination IP `10.10.10.30` belonged to the Ubuntu victim machine.

Port `8000` corresponded with the Python HTTP server running on Ubuntu.

The HTTP User-Agent identifying itself as:

```text
Nmap Scripting Engine
```

also correlated directly with the Nmap reconnaissance activity generated from Kali.

Together, these fields provided strong evidence that the alert represented the controlled service-enumeration activity performed during the lab.

## Analyst Assessment

### Classification

**Suspicious reconnaissance / service enumeration**

### Reasoning

The traffic represented active probing of a network service.

The source system was interacting with the victim's HTTP service and using Nmap scripting functionality to gather information about the target.

This activity was suspicious from a defensive monitoring perspective because reconnaissance can be used by an attacker to identify:

- Running services
- Open ports
- Application technologies
- Operating-system characteristics
- Potential attack surfaces

However, this alert alone did **not** demonstrate that the Ubuntu victim had been compromised.

It demonstrated reconnaissance activity rather than successful exploitation.

## Investigation Pivots

If this activity appeared during a real SOC investigation, I would continue the investigation by pivoting on:

- Source IP address
- Destination IP address
- Destination port
- Community ID
- Other Suricata alerts involving the source
- Related HTTP activity
- Packet capture
- Additional connections from the same source
- Activity occurring immediately before and after the alert

These pivots would help determine whether the reconnaissance was isolated or part of a larger attack sequence.

## Investigation Workflow

```text
Alert Generated
      ↓
Review Alert Metadata
      ↓
Identify Source and Destination
      ↓
Validate Destination Service
      ↓
Review HTTP Evidence
      ↓
Correlate User-Agent with Nmap
      ↓
Review Related Alerts
      ↓
Inspect Supporting Packet Data
      ↓
Determine Activity Context
      ↓
Classify Activity
```

## Conclusion

The investigation confirmed that Security Onion successfully detected reconnaissance traffic generated from Kali Linux against the Ubuntu victim.

The alert metadata, HTTP request, network addresses, destination service, and Nmap User-Agent all correlated with the activity performed during the lab.

I classified the event as **suspicious reconnaissance / service enumeration**, with no evidence from this alert alone that exploitation or compromise had occurred.

This exercise demonstrated how individual IDS alerts can be validated by correlating network metadata, application-layer evidence, known system roles, and the surrounding activity rather than evaluating an alert signature in isolation.
