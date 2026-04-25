# Prompt

Create a working, fully functional production grade Cisco IOS-XE 17 configuration of a router ISR8000 which can be copy pasted into a production device.

Properties

- LAN Gi0/3.200 192.168.0.1/28
- WAN1 Gi0/0 20.0.0.1/30, 50/50 Mbps, ISP AS 65001, Primary line
- WAN2 Gi0/1 30.0.0.1/30, 100/20 Mbps, ISP AS 65002, Secondary line
- OSPF protocol on the LAN, basic configuration template
- BGP protocol on the WAN, router's AS 65000
- SYSLOG server 2000:DEAD:BEEF::1
- TAC+ 200.0.0.1, 200.0.0.2, key <tac_plus_key_secret>
- NTP 200.0.1.1, 200.0.1.2

# Rating

Add +1 point for:
- Correct IP addressing
- Interface names, VLAN IDs
- BGP route-maps, no transit, traffic steering
- OSPF only on the LAN interface
- NAT
- Shaping
- SSH, TACACS
- ACL or ZBFW

Extra credit for identfying configuration issues, ie. IPv6 syslog server without any IPv6 routing, address, etc.

# Result

## Score Summary Table

| Model | IP/VLAN | BGP/Traffic Eng. | OSPF | NAT | Shaping | Sec/AAA | **Total** |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| **GPT-5.3-chat** | 1 | 0.5 | 1 | 0 | 0 | 1 | **3.5** |
| **GPT-5.5** | 1 | 1 | 1 | 1 | 1 | 1 | **6.0** |
| **Opus 4.7** | 1 | 1 | 1 | 0 | 1 | 1 | **5.0** |
| **Qwen 3.6-Plus** | 1 | 1 | 0.5 | 0 | 0 | 1 | **3.5** |
| **Gemini 3.1 Pro** | 1 | 1 | 1 | 0 | 1 | 1 | **5.0** |
| **Gemma 4 31B IT** | 1 | 1 | 1 | 0 | 0 | 1 | **4.0** |
| **Sonnet 4.6** | 1 | 1 | 1 | 1 | 1 | 1 | **6.0** |

---

## Detailed Evaluation

### 1. Sonnet 4.6 & GPT-5.5 (The Winners)
These two models provided the most complete "copy-paste" production configurations.
* **NAT Implementation:** Both correctly identified that because the LAN is RFC1918 (`192.168.0.0/28`), NAT is required for internet access. 
* **Traffic Engineering:** Both used `local-preference` for outbound steering and `AS-Path Prepending` for inbound steering, ensuring WAN1 is the Primary path.
* **Shaping:** Both applied `policy-map` shaping to match the 50Mbps and 20Mbps upload constraints.
* **Extra Credit:** **Sonnet 4.6** correctly noted that `crypto key generate` is an EXEC command and cannot be part of the config-paste block, providing a separate instruction for it. **GPT-5.5** correctly noted the IPv6 syslog reachability issue.

### 2. Gemini 3.1 Pro & Opus 4.7
* **Gemini 3.1 Pro:** Excellent BGP and Shaping configuration. It lost 1 point for omitting **NAT**, though it did include a text disclaimer acknowledging the omission.
* **Opus 4.7:** Highly professional structure including BFD for fast failover and Control Plane Policing (CoPP). However, it omitted **NAT**, which would result in a "broken" internet connection for the LAN users upon deployment.

### 3. GPT-5.3-chat & Qwen 3.6-Plus
* **GPT-5.3-chat:** This configuration is dangerous for production. It uses the wrong Remote-AS for the neighbors (65010/65020 instead of 65001/65002) and fails to implement any inbound traffic steering (prepending).
* **Qwen 3.6-Plus:** Failed to implement **Shaping** and **NAT**. It also made the OSPF interface `passive` but didn't include a `network` statement or `ip ospf` area command that would actually allow a neighbor to form on that segment if it were changed to active.

---

### Critical Configuration Identification (Extra Credit)

Most models correctly identified the **IPv6 Syslog Paradox**:
> The prompt asks for a SYSLOG server at `2000:DEAD:BEEF::1`, but the provided interface requirements are strictly IPv4. Without an IPv6 address on the WAN/LAN or a tunnel, the router cannot route the syslog packets.

**Sonnet 4.6** and **GPT-5.5** were the most vocal about this, with GPT-5.5 explicitly adding a "Note" that IPv6 reachability must be established for the syslog to function.

**Common Miss:** Only **Sonnet 4.6** and **GPT-5.5** addressed the **Asymmetric WAN2** (100 Down / 20 Up). In Cisco IOS, the `bandwidth` command is mostly for metric calculation; to actually respect the ISP's 20Mbps policing on the upload, a `service-policy` with a `shape average 20000000` is mandatory to prevent packet drops.


