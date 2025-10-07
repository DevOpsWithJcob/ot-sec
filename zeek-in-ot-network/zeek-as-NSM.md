## Key benefits in OT
Highlights Zeek's ability to provide visibility into converged IT/OT networks, support threat hunting, and integrate with tools like Elastic Security for alerting on current trends such as ransomware and supply chain attacks targeting critical infrastructure.

## Installation methods


## Configuration Files

**node.cfg**: Defines the deployment topology, such as standalone or cluster modes; key settings include node type (e.g., worker), host/IP, and interface for capturing OT traffic—example: 
```
[worker] type=worker host=localhost interface=eth0.
```


**networks.cfg**: Specifies monitored subnets; crucial for OT to focus on ICS segments—example: add lines like 192.168.1.0/24 to limit analysis to control system networks.

 **zeekctl.cfg**: Controls operational parameters like mail alerts or log rotation; adjust MailTo for notifications on detected anomalies in ICS protocols.

 **site/local.zeek**: Custom script loading file; load ICS-specific analyzers here—example: @load protocols/modbus to enable Modbus logging, or add custom scripts for anomaly detection.

 Additional files: Use loaded_scripts.log to verify configurations; for OT, ensure scripts for protocols like DNP3 are loaded to generate protocol-specific logs without overwhelming storage.

---

**Focus on ICS Attacks**: Emphasizes detection of threats like unauthorized commands, protocol anomalies, and unusual patterns in OT protocols (e.g., Modbus, DNP3). Covers current trends such as ransomware targeting critical infrastructure and supply chain vulnerabilities.

- **Key Benefits in OT**: Provides deep visibility into converged IT/OT networks, supports threat hunting, and integrates with tools like Elastic Security. Helps detect subtle attacks without impacting industrial processes.
- **Learning Outcomes**: Attendees will be equipped to deploy Zeek, customize it for ICS protocols, and use it for proactive defense against evolving threats.

---
## Implementation steps

**Prepare**
- install zeek
- setup monitoring hardware: Use network taps or SPAN ports to capture traffic passively, avoiding any interference with ICS devices like PLCs or SCADA systems.

**Basic Standalone Setup**:
- Edit node.cfg to define a single node (e.g., host: localhost, interface: eth0 for OT traffic capture).
- Run zeekctl deploy to start monitoring. Test with a simple command: zeek -i eth0.

**Cluster Deployment for Larger Networks**:
- Configure multiple nodes in node.cfg (e.g., a logger for log management and workers for analysis).
- Distribute across hosts for scalability in segmented OT environments, such as separate zones for manufacturing floors.

**Running on Live Traffic**:

- Use zeek -i interface-name -C to ignore checksum errors (common in OT due to offloaded packets).
- Integrate with packet capture tools like tcpdump for initial testing.

**Testing with PCAP Files**
- Process files with zeek -r <file.pcap> to simulate scenarios.
- Enable JSON output: Add redef LogAscii::use_json = T; in scripts for easier SIEM integration.

**Integration Tips**:
- Place Zeek sensors in the OT DMZ to monitor without affecting operations.
- Combine with firewalls or other IDS for layered security. Regularly test in a lab environment to mimic real OT setups.

---
## Explanation of Configuration Functions

Configurations in Zeek allow fine-tuning for ICS detection. Here's how key functions work, explained simply.

- **Protocol Loading**: Use @load in local.zeek to activate analyzers.
    - Example: @load protocols/dnp3 – Parses DNP3 traffic, logging function codes for detecting unauthorized reads/writes in power systems.
- **Event Handling**: Define custom responses to events.
    - Example: event modbus_message(c: connection, headers: ModbusHeaders) – Triggers alerts on specific Modbus events, like excessive requests indicating a DoS attack.
- **Log Customization**: Modify log formats and rotations.
    - Example: redef Log::default_rotation_interval = 1day – Rotates logs daily to manage storage in continuous OT monitoring.
- **Threshold Settings**: Set detection limits.
    - Example: const modbus_max_requests = 100 &redef; – Flags potential floods on Modbus devices, common in reconnaissance attacks.
- **Site-Specific Tweaks**: Define local networks.
    - Example: Adjust Site::local_nets to identify internal OT traffic vs. external threats.

These functions make Zeek adaptable to OT's unique needs, like handling proprietary protocols.


---
## Interesting Examples and Scenarios for Engagement

To keep the presentation engaging, incorporate hands-on examples and real-world stories. These illustrate Zeek's practical value.

- **Scenario 1: Modbus Exploitation in Water Treatment**:
    - Attacker sends unauthorized write commands to alter pump settings.
    - Zeek Detection: modbus.log shows unusual function codes; demo pivoting to source IP for quick response.
- **Scenario 2: DNP3 Reconnaissance in Power Grids**:
    - Attacker queries device status repeatedly.
    - Zeek Detection: dnp3.log flags unsolicited responses; hands-on: Script alerts for thresholds like minimum replies.
- **Real-World Example: Stuxnet-Like Attack on PLCs**:
    - Malware uses S7comm for propagation (as in the 2010 Stuxnet incident).
    - How Zeek Helps: Detects anomalous block transfers in s7comm.log; early deployment could have alerted to unusual network activity.
- **Engaging Demo: Brute-Force Telnet on ICS Devices**:
    - Replay a PCAP of an attack; Zeek's weird.log flags invalid methods.
    - Tie to Trends: Discuss IT/OT convergence vulnerabilities, like ransomware via SMB.
- **Interactive Element**: Poll the audience on common OT threats (e.g., ransomware). Demonstrate Zeek detecting lateral movement in conn.log, then discuss custom scripting.