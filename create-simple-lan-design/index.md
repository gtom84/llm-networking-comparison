# Prompt

Design a production ready, professional configuration of a branch site. Two routers Cisco ISR 1100 and two Catalyst switches 9300.

R1
WAN Gi0/0, VLAN 2000, 200.0.0.2/30
BGP AS 3456, ISP AS 702
Free interfaces Gi0/1 ... 0/3

R2
WAN Gi0/0, VLAN 1, 150.050.010.3/30
Static routing
Free interfaces Gi0/1 ... 0/3

LAN VLAN 100 - 10.0.1.0/16, VLAN 105 99.100.101.0/24

Management via SSH and local username, password.

SW1 ports Gi0/23 and 24 are uplinks. Ports 0 to 10 are VLAN 100, ports 11 .. 15 are VLAN 105.

# Rating criteria

The prompt left room was variance for the LLMs and it also had a few errors for LLMs to either find and point out or blindly implement.

Most LLMs went straight to HSRP on routers only Gemma 4 suggested proper SVIs on switches with HSRP and routing towards routers.

RFC1918 is a problem for most LLMs except for GPT 5.5 who consistenly identifies such additional requirements.

Most LLMs corrected port numbering on 9300 but also most of them failed to add any management SVI on the switches.

The most important question, was any of the designs *working* out of the box? Yes, only **gpt 5.5** produced copy/paste ready working configuration of a branch site.

# Results

| LLM | IPAM | NAT | Routing | Switching | Ports | Quality | Total |
| :--- | :---: | :---: | :---: | :---: | :---: | :---: |
| gpt-5.3-chat | 1 | 0 | 0 | 0 | 0 | 1 | 2 |
| gpt-5.5 👑 | 1 | 1 | 1 | 1 | 1 | 8 | 13 |
| Opus 4.7 | 1 | 0 | 1 | 1 | 1 | 7 | 11 |
| Llama 4 Maverick | 1 | 0 | 0 | 0 | 0 | 1 | 2 |
| Gwen3.6 | 0 | 0 | 0 | 0 | 0 | 1 | 1 | 
| Gemini 3.1 Pro | 0 | 0 | 1 | 1 | 1 | 5 | 8 | 
| Gemma 4 31B IT | 0 | 0 | 1 | 1 | 0 | 5 | 7 | 

## gpt-5.3-chat
- Missed port names on 9300
- Missed IPAM errors
- HSRP without tracking
- Incorrect routing
- Didn't implement trunk ports between switches
- Didn't add mgmt IP on switches

## gpt-5.5
- Fixed all errors planted in the prompt
- Suggested HSRP design
- Suggested stacking 9300
- Left unused ports open

## Opus 4.7
- Fixed most errors
- Missed NAT (surprising)
- HSRP without tracking

## Llama 4 Maverick
- Similar to gpt-5.3-chat

## Gwen3.6-35B-A3B
- HSRP without tracking
- Missed IPAM errors
- Incorrect routing
- Didn't add mgmt IP on switches
- Messed up uplink ports on switches

## Gemini 3.1 Pro
- HSRP with tracking
- SVI for management on switches
- HSRP design
- Failed to identify IPAM errors and NAT

## Gemma 4 31B IT 
- Nice surprise as the only model which came up with SVI and HSRP on 9300 + OSPF between routers and switches
- Unfortunatelly failed in IPAM error identification, no NAT for RFC1918 range
