# Comprehensive Guide to Zeek Configurations for Detecting Malicious Activity in OT/ICS Networks

## Overview
Zeek (formerly Bro) is an open-source network security monitoring (NSM) tool widely used in Operational Technology (OT) and Industrial Control Systems (ICS) environments to passively analyze traffic and generate logs that reveal malicious activity. In OT networks, where protocols like Modbus, DNP3, and S7comm are prevalent, Zeek excels at detecting anomalies such as unauthorized commands, malware exploits, and unusual traffic patterns without disrupting operations. This guide explains key configurations, highlights common issues and mistakes, provides solutions, and outlines established rules for effective malicious log detection. It assumes a typical setup with Zeek deployed via zeekctl in standalone or cluster mode, focusing on ICS-specific protocols. Actionable insights are included to optimize detection while minimizing false positives.

## Key Configurations Explained
Proper configuration ensures Zeek captures and analyzes OT traffic effectively, generating logs like conn.log, modbus.log, and weird.log for malicious activity detection.

- **Core Configuration Files**:
  - **node.cfg and networks.cfg**: Define monitoring interfaces and subnets. For OT, specify ICS segments (e.g., 192.168.1.0/24 for PLC networks) to focus on critical traffic and reduce noise. Actionable: Edit networks.cfg to include only OT zones, preventing overload from IT traffic.
  - **site/local.zeek**: Load protocol analyzers and custom scripts. Example: `@load protocols/modbus` and `@load protocols/dnp3` for ICS parsing; add `@load base/frameworks/notice` for alerting. To configure for optimal log detection, ensure threshold settings for alerts (e.g., `redef Notice::default_mailto = "soc@company.com"`) are set to minimize false positives by tuning based on baseline OT traffic.
  - **zeekctl.cfg**: Manage log rotation and telemetry. Set `LogRotationInterval=1hr` for high-volume ICS environments to handle continuous data flows.

- **Protocol-Specific Settings**:
  - Enable deep packet inspection (DPI) for OT protocols to log function codes and payloads. For example, in custom scripts, define events like `event modbus_request(c: connection, ...)` to flag anomalous writes.
  - Integrate with SIEM tools like Elastic Security: Output logs in JSON format (`redef LogAscii::use_json=T`) for correlation with ICS anomalies.

- **Deployment in OT**:
  - Use passive taps or SPAN ports for traffic mirroring. In cluster mode, assign workers to OT chokepoints (e.g., between SCADA and PLCs). Actionable: Start with key segments for visibility, then expand using community plugins for malware detection.

## Common Issues and Mistakes
Deploying Zeek in ICS/OT often encounters challenges due to the unique nature of industrial networks, leading to missed detections or security gaps.

- **Vulnerable Plugins and Outdated Components**: Failing to update Zeek plugins can expose the system to exploits, potentially compromising ICS networks. Common mistake: Neglecting regular software updates, which leaves vulnerabilities to attacks like those exploiting protocol analyzers.
- **Capture Loss and Performance Overload**: Misconfigured interfaces lead to dropped packets, especially in high-throughput OT environments. Mistake: Not accounting for upstream issues in taps/SPAN ports, resulting in incomplete logs.
- **Inadequate Protocol Handling**: Traditional IDS/Zeek setups struggle with proprietary OT protocols, missing subtle anomalies like malware-altered Modbus commands. Common error: Overlooking network segmentation, allowing IT noise to drown OT signals.
- **False Positives from Baselines**: Untuned thresholds generate excessive alerts on normal OT variability (e.g., periodic scans), impacting alert fatigue.
- **Bypassing Controls**: Temporary insecure actions during maintenance bypass Zeek monitoring, a frequent ICS pitfall.

These issues hinder effective management of security protocols, such as delayed detection of ransomware or APTs in converged IT/OT setups.

## Solutions and Actionable Insights
Address issues with targeted fixes to enhance malicious log detection.

- **Secure and Update Components**: Regularly patch Zeek and plugins; use official repositories for secure deployments. Solution: Implement automated update scripts and vulnerability scanning on Zeek sensors to prevent compromise. Actionable: Schedule weekly checks via `zeekctl diag` and integrate with patch management tools.
- **Mitigate Capture Loss**: Verify TAP/SPAN configurations upstream; use `-C` flag in Zeek to ignore checksum errors common in OT. Actionable: Monitor `packet_loss.log` and scale to cluster mode for load balancing in busy ICS segments.
- **Enhance Protocol Parsing**: Develop or load custom Zeek scripts for proprietary protocols; use community extensions for DPI on Modbus/DNP3. Solution: Baseline normal traffic with PCAP replays to tune anomaly detection.
- **Reduce False Positives**: Establish OT baselines and adjust thresholds (e.g., `const max_modbus_requests = 50 &redef;`). Integrate ML for adaptive alerting. Actionable: Use Elastic Security to correlate Zeek logs with ICS telemetry for contextual alerts.
- **Enforce Segmentation and Controls**: Deploy Zeek in DMZs with strict access; log all maintenance bypasses. Solution: Combine with firewalls for hybrid monitoring.

For broader threats like APTs, leverage Zeek's MITRE ATT&CK coverage via scripts for techniques like port knocking or DNS tunneling.

## Established Rules for Malicious Log Detection
Zeek's scripting language enables rule-based detection of malicious patterns in logs. Follow these rules, derived from best practices:
[[Malicious Log Detection]]
- **Anomaly-Based Rules**: Monitor for deviations in protocol logs (e.g., unusual Modbus function codes in modbus.log indicating malware). Rule: If `func=WRITE_MULTIPLE_REGISTERS` exceeds baseline, generate notice via `NOTICE([$note=ModbusAnomaly, ...])`.
- **Signature-Based Rules**: Script for known malicious indicators, like brute-force in ssh.log or suspicious User-Agents in http.log. Rule: Track failed logins >5 in 1min: `event ssh_failed_auth(c: connection) { if (++attempts > 5) NOTICE([...]) }`.
- **Behavioral Rules**: Detect C2 via DNS queries or encrypted traffic anomalies. Rule: Flag high-volume NXDOMAIN responses in dns.log for potential malware callbacks.
- **OT-Specific Rules**: For ICS, rule on unsolicited DNP3 responses or S7comm block transfers. Integrate with threat intel feeds for dynamic blacklisting.
- **General Best Practice Rule**: Always correlate logs (using UID) across conn.log, protocol logs, and weird.log; suppress notices for known benign OT behaviors.

Actionable: Test rules with PCAPs simulating attacks (e.g., Modbus exploits); refine via `notice.log` analysis to align with 2025 threats like state-sponsored ICS intrusions.

## Conclusion
By configuring Zeek with OT-focused protocols, addressing common pitfalls like plugin vulnerabilities and capture issues, and applying scripted rules, organizations can robustly detect malicious activity in logs. Start with baselining your environment, implement the solutions outlined, and regularly review detections to adapt to evolving threats. For hands-on implementation, refer to Zeek documentation and integrate with tools like Elastic for comprehensive ICS security. This approach optimizes log detection while maintaining operational integrity.