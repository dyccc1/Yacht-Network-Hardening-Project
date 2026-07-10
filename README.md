# Superyacht SOC Simulation: Protecting Maritime Infrastructure

## Project Context
As a former yacht crew member, I understand that a superyacht's network is a critical and unique environment: **UHNWI passengers** demand total privacy, while **navigation systems (ECDIS)** cannot be compromised by unauthorized crew access or malware.

This project simulates a luxury yacht's security ecosystem, utilizing **Wazuh SIEM**, **pfSense Firewall**, and **Endpoint Hardening** to ensure bridge integrity and onboard data protection.

## Full Documentation
Detailed technical report with all steps, logs, and screenshots here:

* Download Technical Report - Volume 1 (PDF): [technical_report_vol1.pdf](https://github.com/user-attachments/files/29900829/technical_report_vol1.pdf)
* Download Technical Report - Volume 2 (PDF): [technical_report_vol2.pdf](https://github.com/user-attachments/files/29901431/technical_report_vol2.pdf)
* Download Technical Report - Volume 3 (PDF): [technical_report_vol3.pdf](https://github.com/user-attachments/files/29902676/technical_report_vol3.pdf)


## Network Topology
<img width="580" height="521" alt="Diagrama sem nome drawio (1)" src="https://github.com/user-attachments/assets/2cfc42b0-42e4-4715-bd3b-643dfcca4748" />


---

## Security Implementation (Volume 1)

### 1. Firewall Hardening (pfSense)
I implemented **Micro-segmentation** policies using Aliases to simplify rule management.

<img width="1040" height="681" alt="image" src="https://github.com/user-attachments/assets/5ea77d69-0e25-446e-8fd6-0b0e54b976aa" />

*   **Critical Policy:** Explicit blocking of traffic between the crew network (**CREW**) and the navigation network (**BRIDGE/ECDIS**) with active logging for SIEM auditing.

<img width="705" height="682" alt="image" src="https://github.com/user-attachments/assets/8e5206ff-1445-46c9-bc10-9c4775eaa22b" />

**Implemented Security Zones:**
*   **VLAN 40 (AV CONTROL):** Entertainment Systems.
*   **VLAN 30 (Bridge/ECDIS):** Isolated zone for critical navigation systems. Monitored by Wazuh Agent and Sysmon.
*   **VLAN 20 (Crew):** Segmented network with firewall rules preventing lateral movement to the bridge.
*   **VLAN 10 (Guest):** Internet access only.
*   **WAN (VSAT Link):** Satellite connection simulation monitored via Syslog.

### 2. Detection Intelligence (Wazuh SIEM)
I developed a **Rule Hierarchy (Parent/Child)** to identify the exact origin of network intrusion attempts.

```xml
<group name="yacht_cyber_security,">
 
  <rule id="100030" level="10">
    <if_sid>87701</if_sid>
    <dstip>10.0.30.0/24</dstip>
    <match>22|3389</match>
    <description>pfSense: Tentativa de acesso aos sistemas da PONTE detetada.</description>
    <mitre><id>T1021</id></mitre>
  </rule>

  
  <rule id="100031" level="12">
    <if_matched_sid>100030</if_matched_sid>
    <srcip>10.0.10.0/24</srcip>
    <description>ALERTA CRÍTICO: Tentativa de invasão dos GUESTS à PONTE!</description>
  </rule>

  
  <rule id="100032" level="12">
    <if_matched_sid>100030</if_matched_sid>
    <srcip>10.0.20.0/24</srcip>
    <description>ALERTA CRÍTICO: Tentativa de invasão da TRIPULAÇÃO à PONTE!</description>
  </rule>

  
  <rule id="100033" level="12">
    <if_matched_sid>100030</if_matched_sid>
    <srcip>10.0.40.0/24</srcip>
    <description>ALERTA CRÍTICO: Dispositivo de ENTRETENIMENTO (AV) a tentar aceder à PONTE!</description>
  </rule>
</group>
 ```

### Troubleshooting & Lessons Learned
This lab presented technical challenges that required deep diagnostics, such as:
*   **802.1Q Tag Conflict:** Identified a connectivity failure after activating VLANs; resolved by configuring the LAN interface via CLI to accept untagged management traffic.
*   **XML Syntax Analysis:** Overcame IP mapping errors in the Wazuh analysis engine by adjusting rules to the correct CIDR notation.

---

## WAN Redundancy (Volume 2)

### High Availability at Sea
In maritime operations, the internet is more than just a service, it is a critical utility. The satellite link (**Starlink**) can suffer interruptions due to atmospheric conditions or geographical location. To ensure the yacht never goes offline, I implemented a **Multi-WAN Failover** system in pfSense.


### Infrastructure Configuration
To simulate this environment, I configured the firewall with two redundant gateways:
*   **WAN 1 (Starlink):** Defined as the primary link (Tier 1).
*   **WAN 2 (4G/5G Backup):** Defined as the backup link (Tier 2).


### Technical Implementation
1.  **Gateway Groups:** Created a load balancing/failover group named `yacht_internet_failover`.
2.  **Trigger Mechanism:** Failover is automatically triggered by the "**Member Down**" status. pfSense continuously monitors packet loss and latency from Starlink.
3.  **Policy Routing:** Forced critical VLAN traffic to use this group through firewall rules.


### Proof of concept (POC)
<img width="1173" height="355" alt="image" src="https://github.com/user-attachments/assets/e9d940d5-2187-42be-827d-4dbeadeaeab0" />
<img width="1021" height="663" alt="image" src="https://github.com/user-attachments/assets/86c8b1ad-e0fd-4dbd-acc0-5e3219d7dc7d" />


### Lessons Learned:
*   **External IP Monitoring:** I learned that for effective failover, the firewall must monitor an external IP (such as `8.8.8.8`) rather than just the local Gateway to detect real internet outages beyond "unplugged cables."
*   **Session Persistence:** Correct configuration ensured that during a link switch, critical telemetry communications for **Wazuh** were not interrupted, maintaining 24/7 security visibility.

---

## Site-to-Site VPN (Volume 3)

### Yacht-to-Office Interconnection via WireGuard
To simulate remote vessel management from shore, I implemented an encrypted VPN tunnel connecting the **Yacht** to a secondary pfSense instance representing the **Management Office**.

*   **Protocol:** WireGuard (UDP/51820).
*   **Security:** Asymmetric cryptography based on public/private key exchange.
*   **Objective:** Allow secure access to critical systems (ECDIS/Wazuh) and administrative traffic without exposing ports to the public internet.
<img width="1008" height="789" alt="image" src="https://github.com/user-attachments/assets/0d984b01-7c6f-4e21-a1b5-64964e96d7a5" />

### VPN Troubleshooting: The Handshake Challenge
The biggest technical challenge of this phase was overcoming the `Latest Handshake: Never` status.

*   **Diagnostic:** Through real-time analysis of **Firewall Logs (Dynamic View)**, I identified that pfSense was discarding tunnel packets due to native protection against private networks on the WAN interface.
*   **Resolution:** Disabled the implicit 'Block Private Networks' rule on both sides and configured static routes (**Allowed IPs**) to ensure correct routing between the `10.0.0.0/24` and `192.168.50.0/24` networks.
<img width="959" height="436" alt="image" src="https://github.com/user-attachments/assets/dd9b855b-e2a5-4e9e-9c4e-095a5980970c" />
<img width="1049" height="740" alt="image" src="https://github.com/user-attachments/assets/58a6be04-0fdb-438a-aa64-d881c2342635" />
<img width="1391" height="424" alt="image" src="https://github.com/user-attachments/assets/c0123100-a76f-46e8-836f-7ac137b142e8" />

### Lessons Learned:
1.  **Implicit Perimeter Protections:** Realized pfSense automatically drops traffic from private IP ranges (RFC 1918) on the WAN interface. In a virtual lab environment, disabling "Block Private Networks" was crucial for the VPN handshake.
2.  **Visibility via Firewall Logs:** Learned to use the **Dynamic View** of pfSense logs; seeing the "Red X" on UDP 51820 packets allowed me to identify firewall blocks that the VPN status alone did not explain.
3.  **Asymmetric Cryptography:** The practice of generating and cross-referencing **Public Keys** reinforced the importance of attention to detail: a single wrong character results in total communication silence.
<img width="1631" height="661" alt="image" src="https://github.com/user-attachments/assets/c490f7c9-a04f-40e0-a53d-a47e7b34fef3" />
---

## Upcoming Volumes:
*   [ ] Access control and third-party auditing.
*   [ ] Defense against active eavesdropping/IoT.
*   [ ] Unauthorized device detection (WIPS).
*   [ ] Open Source Intelligence (OSINT) monitoring.
*   [ ] Air-gapped backup and recovery.
*   [ ] Cyber Drills/Incident Response simulations.
*   [ ] Integrated global testing.
