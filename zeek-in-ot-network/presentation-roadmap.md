# Zeek in ICS Security: A Comprehensive Presentation Roadmap for OT Networks

## Introduction
Welcome to this comprehensive roadmap on utilizing Zeek for detecting attacks in Industrial Control Systems (ICS) within Operational Technology (OT) networks. Zeek, formerly known as Bro, is a powerful open-source network security monitoring tool that excels in passive traffic analysis. It's particularly valuable in OT environments where active scanning could disrupt critical operations.

This document is designed for audiences with varying levels of expertise—from beginners in ICS security to seasoned cybersecurity professionals. We'll break down concepts clearly, avoid unnecessary jargon, and include practical examples. By the end, you'll have a solid understanding of how to implement Zeek, configure it for OT-specific needs, and apply it to real-world threat detection.

Key learning outcomes:
- Understand Zeek's role in OT security.
- Learn step-by-step implementation and configuration.
- Explore engaging scenarios and real-world attack examples.
- Gain guidance on logging, detection, and resources like PCAP files.

---

## 1. Roadmap Overview
This presentation roadmap provides a structured path to mastering Zeek in ICS attack detection. It starts with foundational knowledge and progresses to advanced applications, ensuring accessibility for all.

- **Introduction to Zeek as a Network Security Monitoring (NSM) Tool**: Zeek passively analyzes network traffic to generate detailed logs on connections, protocols, and anomalies. Unlike traditional IDS, it focuses on visibility rather than blocking, making it ideal for sensitive OT networks where downtime is unacceptable.
- **Focus on ICS Attacks**: Emphasizes detection of threats like unauthorized commands, protocol anomalies, and unusual patterns in OT protocols (e.g., Modbus, DNP3). Covers current trends such as ransomware targeting critical infrastructure and supply chain vulnerabilities.
- **Presentation Structure**: 
  - Foundational concepts for beginners.
  - Practical implementation and configurations.
  - Real-world scenarios for engagement.
  - Advanced guidance on logs, detection, and resources.
- **Key Benefits in OT**: Provides deep visibility into converged IT/OT networks, supports threat hunting, and integrates with tools like Elastic Security. Helps detect subtle attacks without impacting industrial processes.
- **Learning Outcomes**: Attendees will be equipped to deploy Zeek, customize it for ICS protocols, and use it for proactive defense against evolving threats.

---

## 2. Implementation Steps
Implementing Zeek in OT networks requires careful planning to ensure non-disruptive monitoring. Follow these steps for a smooth rollout.

1. **Preparation**:
   - Install Zeek using pre-built binaries (easiest for most users) or build from source on Unix systems. Check the official Zeek documentation for your platform.
   - Set up monitoring hardware: Use network taps or SPAN ports to capture traffic passively, avoiding any interference with ICS devices like PLCs or SCADA systems.

2. **Basic Standalone Setup**:
   - Edit `node.cfg` to define a single node (e.g., host: localhost, interface: eth0 for OT traffic capture).
   - Run `zeekctl deploy` to start monitoring. Test with a simple command: `zeek -i eth0`.

3. **Cluster Deployment for Larger Networks**:
   - Configure multiple nodes in `node.cfg` (e.g., a logger for log management and workers for analysis).
   - Distribute across hosts for scalability in segmented OT environments, such as separate zones for manufacturing floors.

4. **Running on Live Traffic**:
   - Use `zeek -i <interface> -C` to ignore checksum errors (common in OT due to offloaded packets).
   - Integrate with packet capture tools like tcpdump for initial testing.

5. **Testing with PCAP Files**:
   - Process files with `zeek -r <file.pcap>` to simulate scenarios.
   - Enable JSON output: Add `redef LogAscii::use_json = T;` in scripts for easier SIEM integration.

6. **Integration Tips**:
   - Place Zeek sensors in the OT DMZ to monitor without affecting operations.
   - Combine with firewalls or other IDS for layered security. Regularly test in a lab environment to mimic real OT setups.

---

## 3. Configuration Files
Zeek's flexibility comes from its configuration files. Here's a breakdown of key files and how to use them in OT contexts.

- **node.cfg**: Defines deployment topology.
  - Example: `[worker] type=worker host=localhost interface=eth0` – Captures traffic from OT interfaces.
  - Tip: Use for standalone or cluster modes in segmented ICS networks.

- **networks.cfg**: Specifies monitored subnets.
  - Example: Add `192.168.1.0/24` to focus on ICS segments.
  - Benefit: Reduces noise by limiting analysis to critical OT areas.

- **zeekctl.cfg**: Manages operational settings.
  - Adjust `MailTo` for email alerts on ICS anomalies.
  - Set log rotation intervals to handle high-volume OT traffic.

- **site/local.zeek**: Loads custom scripts.
  - Example: `@load protocols/modbus` to enable Modbus analysis.
  - Customize: Add scripts for OT-specific protocols like DNP3 or S7comm.

- **Additional Files**:
  - `loaded_scripts.log`: Verify loaded configurations.
  - For OT: Ensure only essential scripts are loaded to optimize performance on resource-constrained devices.

---

## 4. Explanation of Configuration Functions
Configurations in Zeek allow fine-tuning for ICS detection. Here's how key functions work, explained simply.

- **Protocol Loading**: Use `@load` in `local.zeek` to activate analyzers.
  - Example: `@load protocols/dnp3` – Parses DNP3 traffic, logging function codes for detecting unauthorized reads/writes in power systems.

- **Event Handling**: Define custom responses to events.
  - Example: `event modbus_message(c: connection, headers: ModbusHeaders)` – Triggers alerts on specific Modbus events, like excessive requests indicating a DoS attack.

- **Log Customization**: Modify log formats and rotations.
  - Example: `redef Log::default_rotation_interval = 1day` – Rotates logs daily to manage storage in continuous OT monitoring.

- **Threshold Settings**: Set detection limits.
  - Example: `const modbus_max_requests = 100 &redef;` – Flags potential floods on Modbus devices, common in reconnaissance attacks.

- **Site-Specific Tweaks**: Define local networks.
  - Example: Adjust `Site::local_nets` to identify internal OT traffic vs. external threats.

These functions make Zeek adaptable to OT's unique needs, like handling proprietary protocols.

---

## 5. Interesting Examples and Scenarios for Engagement
To keep the presentation engaging, incorporate hands-on examples and real-world stories. These illustrate Zeek's practical value.

- **Scenario 1: Modbus Exploitation in Water Treatment**:
  - Attacker sends unauthorized write commands to alter pump settings.
  - Zeek Detection: `modbus.log` shows unusual function codes; demo pivoting to source IP for quick response.

- **Scenario 2: DNP3 Reconnaissance in Power Grids**:
  - Attacker queries device status repeatedly.
  - Zeek Detection: `dnp3.log` flags unsolicited responses; hands-on: Script alerts for thresholds like minimum replies.

- **Real-World Example: Stuxnet-Like Attack on PLCs**:
  - Malware uses S7comm for propagation (as in the 2010 Stuxnet incident).
  - How Zeek Helps: Detects anomalous block transfers in `s7comm.log`; early deployment could have alerted to unusual network activity.

- **Engaging Demo: Brute-Force Telnet on ICS Devices**:
  - Replay a PCAP of an attack; Zeek's `weird.log` flags invalid methods.
  - Tie to Trends: Discuss IT/OT convergence vulnerabilities, like ransomware via SMB.

- **Interactive Element**: Poll the audience on common OT threats (e.g., ransomware). Demonstrate Zeek detecting lateral movement in `conn.log`, then discuss custom scripting.

---

## 6. Using Zeek in OT Networks
OT networks differ from IT—focus on reliability over speed. Zeek fits perfectly for passive monitoring.

- **Deployment Considerations**: Position sensors at chokepoints (e.g., between PLCs and SCADA) using taps.
- **Supported Protocols**: Built-in for Modbus, DNP3, BACnet, CIP, IEC 104, OPC UA, PROFINET, S7comm; extend with packages like Corelight's ICS/OT Collection.
- **Customization**: Load OT-specific analyzers; integrate with SIEM for alerts on baseline deviations.
- **Challenges & Solutions**: For air-gapped networks, use offline PCAPs. Optimize for low resources by focusing on key protocols.
- **Best Practices**: Update scripts regularly; align with trends like ransomware (e.g., Colonial Pipeline 2021) by detecting C2 over ICS protocols.

---

## 7. Capturing Logs Effectively
Logs are Zeek's core output—make them actionable for OT forensics.

- **Generation Basics**: Protocol-specific logs (e.g., `modbus.log`) in TSV/JSON; use `zeekctl` for management.
- **Optimization**: Load only needed analyzers; prefix logs (e.g., `redef Log::default_ext_prefix = "ot_"`).
- **Storage & Retrieval**: Centralize logs; use `zcat` for compressed files; correlate via unique IDs (uid).
- **Analysis Tips**: Enrich with GeoIP; focus on `weird.log` for anomalies.
- **Integration**: Feed to SIEM for real-time OT alerts, ensuring timestamps for traceability.

---

## 8. Utilizing Zeek for Attack Detection
Zeek excels at spotting subtle ICS threats through traffic parsing.

- **Core Mechanism**: Logs events for anomaly detection (e.g., unauthorized Modbus writes).
- **Examples**: Flag brute-force (via `notice.log`); detect malware like Triton in `conn.log`.
- **Real-World Application**: In EKANS ransomware (2020), Zeek could spot process-killing over ICS; use thresholds for alerts.
- **Advanced Scripting**: Event handlers for DNP3 unsolicited messages; map to MITRE ATT&CK OT tactics.
- **Trends**: Detect supply chain attacks (e.g., SolarWinds) via DNS in `dns.log`; add ML for baselines.

---

## 9. Related PCAP Sample Files
Practice with these free resources to test Zeek in simulated OT attacks.

- **CIC Modbus Dataset 2023**: From unb.ca/cic/datasets – Includes DoS and reconnaissance for Modbus.
- **DNP3 Intrusion Detection Dataset**: ieee-dataport.org – Attacks like spoofing; great for DNP3 logs.
- **Netresec Public PCAPs**: netresec.com (SCADA/ICS section) – Free Modbus/DNP3 files with malware.
- **Malcolm-PCAP ICS Collection**: gitlab.cs.uni-magdeburg.de – OT-specific with potential threats.
- **ELECTRON DNP3 Dataset**: electron-project.eu – Intrusion scenarios for DNP3.
- **Additional**: GitHub's awesome-industrial-protocols for more repositories.

---

## 10. Sample Logs with Explanations
Understand Zeek logs through these examples, focusing on OT relevance.

- **Modbus Log (modbus.log)**:
  - Sample: `ts=1728211200 uid=C123456789 id.orig_h=192.168.1.10 id.orig_p=502 id.resp_h=192.168.1.20 id.resp_p=502 func=WRITE_SINGLE_REGISTER exception= pdus=[address=1, value=0xFF00]`.
  - Explanation: Captures a write command. Detect anomalies if "func" is unauthorized, e.g., altering PLCs in attacks.

- **DNP3 Log (dnp3.log)**:
  - Sample: `ts=1728211200 uid=D987654321 id.orig_h=10.0.0.1 id.orig_p=20000 id.resp_h=10.0.0.2 id.resp_p=20000 iin=0x0000 func_code=READ objects=[group=1, variation=2, points=1-5]`.
  - Explanation: Logs a read request. Flag if "iin" indicates unsolicited, signaling reconnaissance.

- **Weird Log (weird.log)**:
  - Sample: `ts=1728211200 uid=W456789012 name=unknown_modbus_function addl=0x99 peer=analyzer notice=F source=MODBUS`.
  - Explanation: Highlights protocol errors, useful for malformed packets in ICS exploits.

- **Conn Log (conn.log)**:
  - Sample: `ts=1728211200 uid=C1122334455 id.orig_h=192.168.1.5 id.orig_p=12345 id.resp_h=192.168.1.6 id.resp_p=502 proto=tcp service=modbus duration=0.5s orig_bytes=100 resp_bytes=50 history=ShAdDaTf local_orig=T missed_bytes=0`.
  - Explanation: Summarizes connections. Spot attacks via unusual "duration" or "bytes," e.g., data exfiltration.

---

## Conclusion
This roadmap equips you to leverage Zeek effectively in OT for ICS attack detection. Start small with basic setups, experiment with PCAPs, and scale to custom scripts. For more resources, visit zeek.org or explore OT security communities. Stay vigilant—OT threats evolve, but tools like Zeek provide essential visibility. If you have questions, dive deeper into the examples!