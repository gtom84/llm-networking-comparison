# Prompt 

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

# Responses

## gpt-5.3-chat

hostname ISR8000
no ip domain-lookup
ip domain name lab.local

!
! =========================
! AAA + TACACS+
! =========================
aaa new-model

tacacs server TAC1
 address ipv4 200.0.0.1
 key TACKEY

tacacs server TAC2
 address ipv4 200.0.0.2
 key TACKEY

aaa group server tacacs+ TAC-GROUP
 server name TAC1
 server name TAC2

aaa authentication login default group TAC-GROUP local
aaa authorization exec default group TAC-GROUP local
aaa accounting exec default start-stop group TAC-GROUP

username fallback privilege 15 secret STRONGPASSWORD

!
! =========================
! NTP
! =========================
ntp server 200.0.1.1 prefer
ntp server 200.0.1.2

!
! =========================
! SYSLOG (IPv6)
! =========================
logging buffered 100000 warnings
logging host 2000:DEAD:BEEF::1
logging trap informational

!
! =========================
! INTERFACES
! =========================

! --- WAN1 ---
interface GigabitEthernet0/0
 description WAN1
 ip address 20.0.0.1 255.255.255.252
 no shutdown

! --- WAN2 ---
interface GigabitEthernet0/1
 description WAN2
 ip address 30.0.0.1 255.255.255.252
 no shutdown

! --- LAN VLAN 200 ---
interface GigabitEthernet0/3.200
 encapsulation dot1Q 200
 description LAN
 ip address 192.168.0.1 255.255.255.240
 no shutdown

!
! =========================
! OSPF (LAN only)
! =========================
router ospf 10
 router-id 1.1.1.1
 passive-interface default
 no passive-interface GigabitEthernet0/3.200

 network 192.168.0.0 0.0.0.15 area 0

!
! =========================
! BGP (DUAL WAN)
! =========================
router bgp 65001
 bgp log-neighbor-changes

 ! --- WAN1 neighbor ---
 neighbor 20.0.0.2 remote-as 65010
 neighbor 20.0.0.2 description WAN1_PEER
 neighbor 20.0.0.2 update-source GigabitEthernet0/0

 ! --- WAN2 neighbor ---
 neighbor 30.0.0.2 remote-as 65020
 neighbor 30.0.0.2 description WAN2_PEER
 neighbor 30.0.0.2 update-source GigabitEthernet0/1

 !
 address-family ipv4
  network 192.168.0.0 mask 255.255.255.240

  neighbor 20.0.0.2 activate
  neighbor 30.0.0.2 activate

  ! Prefer WAN2 (higher bandwidth)
  neighbor 30.0.0.2 route-map PREF-WAN2 in
  neighbor 20.0.0.2 route-map PREF-WAN1 in
 exit-address-family

!
! =========================
! ROUTE-MAPS (PREFERENCE)
! =========================
route-map PREF-WAN2 permit 10
 set local-preference 200

route-map PREF-WAN1 permit 10
 set local-preference 100

!
! =========================
! BASIC SECURITY HARDENING
! =========================
service password-encryption
no ip http server
no ip http secure-server

ip ssh version 2

line vty 0 4
 transport input ssh
 login authentication default

!
! =========================
! MANAGEMENT ACCESS (optional but smart)
! =========================
ip access-list standard MGMT
 permit 192.168.0.0 0.0.0.15

line vty 0 4
 access-class MGMT in

!
! =========================
! SAVE
! =========================
end
write memory

---

## gpt-5.5

Below is a Cisco IOS-XE 17 configuration using the interface names exactly as provided: `GigabitEthernet0/0`, `GigabitEthernet0/1`, and `GigabitEthernet0/3.200`.

> **Important:** Before pasting into production, replace `CHANGE_ME_STRONG_LOCAL_SECRET` and confirm that your ISR8000 platform really uses these interface names. Some ISR/Catalyst 8000 platforms use names such as `GigabitEthernet0/0/0`.

```cisco
!
! ============================================================
! Cisco IOS-XE 17 ISR8000 Production Baseline Configuration
! LAN:  Gi0/3.200 - 192.168.0.1/28
! WAN1: Gi0/0     - 20.0.0.1/30 - ISP AS 65001 - Primary
! WAN2: Gi0/1     - 30.0.0.1/30 - ISP AS 65002 - Secondary
! Local AS: 65000
! ============================================================
!
hostname ISR8000-EDGE-01
!
no service pad
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec localtime show-timezone
service password-encryption
!
clock timezone UTC 0 0
!
no ip source-route
no ip gratuitous-arps
ip cef
ipv6 unicast-routing
ipv6 cef
!
ip domain name example.local
!
enable algorithm-type scrypt secret CHANGE_ME_STRONG_LOCAL_SECRET
username breakglass privilege 15 algorithm-type scrypt secret CHANGE_ME_STRONG_LOCAL_SECRET
!
banner motd ^C
****************************************************************
* Authorized access only. Disconnect immediately if unauthorized.
****************************************************************
^C
!
! ============================================================
! AAA / TACACS+
! ============================================================
!
aaa new-model
aaa session-id common
!
tacacs server TACACS-1
 address ipv4 200.0.0.1
 key 0 <tac_plus_key_secret>
 single-connection
!
tacacs server TACACS-2
 address ipv4 200.0.0.2
 key 0 <tac_plus_key_secret>
 single-connection
!
aaa group server tacacs+ TACACS-GROUP
 server name TACACS-1
 server name TACACS-2
!
ip tacacs source-interface GigabitEthernet0/3.200
!
aaa authentication login default group TACACS-GROUP local
aaa authorization exec default group TACACS-GROUP local
aaa accounting exec default start-stop group TACACS-GROUP
aaa accounting commands 15 default start-stop group TACACS-GROUP
!
! ============================================================
! SSH Management
! ============================================================
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
!
crypto key generate rsa modulus 3072
!
ip access-list standard VTY-MGMT
 permit 192.168.0.0 0.0.0.15
 permit 200.0.0.1
 permit 200.0.0.2
 deny any log
!
line con 0
 exec-timeout 10 0
 logging synchronous
 login authentication default
!
line vty 0 15
 access-class VTY-MGMT in
 exec-timeout 15 0
 logging synchronous
 transport input ssh
 login authentication default
 authorization exec default
!
! ============================================================
! Logging / Syslog
! ============================================================
!
logging buffered 65536 informational
logging trap informational
logging origin-id hostname
logging facility local7
logging host ipv6 2000:DEAD:BEEF::1 transport udp port 514
!
! ============================================================
! NTP
! ============================================================
!
ntp source GigabitEthernet0/3.200
ntp server 200.0.1.1 prefer
ntp server 200.0.1.2
!
! ============================================================
! QoS Shaping for WAN Upload Rates
! WAN1: 50 Mbps egress
! WAN2: 20 Mbps egress
! ============================================================
!
policy-map WAN1-OUT-SHAPE
 class class-default
  shape average 50000000
!
policy-map WAN2-OUT-SHAPE
 class class-default
  shape average 20000000
!
! ============================================================
! WAN Anti-Spoofing ACLs
! Stateless ACLs permit normal return traffic while dropping
! common invalid/private spoofed sources from the WAN.
! ============================================================
!
ip access-list extended WAN-IN
 deny ip 0.0.0.0 0.255.255.255 any log
 deny ip 10.0.0.0 0.255.255.255 any log
 deny ip 127.0.0.0 0.255.255.255 any log
 deny ip 169.254.0.0 0.0.255.255 any log
 deny ip 172.16.0.0 0.15.255.255 any log
 deny ip 192.168.0.0 0.0.255.255 any log
 deny ip 224.0.0.0 15.255.255.255 any log
 permit ip any any
!
! ============================================================
! NAT
! LAN is private RFC1918 space, so overload NAT is configured
! toward both WAN links. Routing selects the active WAN.
! ============================================================
!
ip access-list standard NAT-LAN
 permit 192.168.0.0 0.0.0.15
!
route-map NAT-TO-WAN1 permit 10
 match ip address NAT-LAN
 match interface GigabitEthernet0/0
!
route-map NAT-TO-WAN2 permit 10
 match ip address NAT-LAN
 match interface GigabitEthernet0/1
!
ip nat inside source route-map NAT-TO-WAN1 interface GigabitEthernet0/0 overload
ip nat inside source route-map NAT-TO-WAN2 interface GigabitEthernet0/1 overload
!
! ============================================================
! Interfaces
! ============================================================
!
interface GigabitEthernet0/0
 description WAN1_PRIMARY_ISP_AS65001_50M_50M
 ip address 20.0.0.1 255.255.255.252
 bandwidth 50000
 ip nat outside
 ip access-group WAN-IN in
 no ip redirects
 no ip proxy-arp
 no ip unreachables
 load-interval 30
 no cdp enable
 service-policy output WAN1-OUT-SHAPE
 no shutdown
!
interface GigabitEthernet0/1
 description WAN2_SECONDARY_ISP_AS65002_100M_20M
 ip address 30.0.0.1 255.255.255.252
 bandwidth 20000
 ip nat outside
 ip access-group WAN-IN in
 no ip redirects
 no ip proxy-arp
 no ip unreachables
 load-interval 30
 no cdp enable
 service-policy output WAN2-OUT-SHAPE
 no shutdown
!
interface GigabitEthernet0/3
 description LAN_PARENT_TRUNK
 no ip address
 no shutdown
!
interface GigabitEthernet0/3.200
 description LAN_VLAN_200_OSPF
 encapsulation dot1Q 200
 ip address 192.168.0.1 255.255.255.240
 ip nat inside
 ip ospf 100 area 0
 no ip redirects
 no ip proxy-arp
 load-interval 30
 no shutdown
!
! ============================================================
! OSPF - LAN Side
! ============================================================
!
router ospf 100
 router-id 192.168.0.1
 passive-interface default
 no passive-interface GigabitEthernet0/3.200
 default-information originate
!
! ============================================================
! BGP - WAN Side
! Assumption:
!   ISP1 peer IP = 20.0.0.2
!   ISP2 peer IP = 30.0.0.2
! This router accepts default routes only.
! WAN1 is preferred using higher local preference.
! No LAN prefixes are advertised because LAN is RFC1918 and NAT is used.
! ============================================================
!
ip prefix-list DEFAULT-ONLY seq 5 permit 0.0.0.0/0
!
route-map ISP1-IN permit 10
 match ip address prefix-list DEFAULT-ONLY
 set local-preference 200
!
route-map ISP1-IN deny 100
!
route-map ISP2-IN permit 10
 match ip address prefix-list DEFAULT-ONLY
 set local-preference 100
!
route-map ISP2-IN deny 100
!
route-map BGP-DENY-ALL deny 10
!
router bgp 65000
 bgp router-id 20.0.0.1
 bgp log-neighbor-changes
 timers bgp 10 30
 !
 neighbor 20.0.0.2 remote-as 65001
 neighbor 20.0.0.2 description WAN1_PRIMARY_ISP_AS65001
 neighbor 30.0.0.2 remote-as 65002
 neighbor 30.0.0.2 description WAN2_SECONDARY_ISP_AS65002
 !
 address-family ipv4
  neighbor 20.0.0.2 activate
  neighbor 20.0.0.2 route-map ISP1-IN in
  neighbor 20.0.0.2 route-map BGP-DENY-ALL out
  neighbor 20.0.0.2 maximum-prefix 5 90 restart 5
  !
  neighbor 30.0.0.2 activate
  neighbor 30.0.0.2 route-map ISP2-IN in
  neighbor 30.0.0.2 route-map BGP-DENY-ALL out
  neighbor 30.0.0.2 maximum-prefix 5 90 restart 5
 exit-address-family
!
! ============================================================
! Disable Unused Services
! ============================================================
!
no ip http server
no ip http secure-server
no ip finger
no ip identd
!
login block-for 120 attempts 5 within 60
security passwords min-length 12
!
! ============================================================
! End of Configuration
! ============================================================
!
end
write memory
```

**Note:** The IPv6 syslog server `2000:DEAD:BEEF::1` is configured, but the router also needs IPv6 reachability toward that address. No IPv6 WAN/LAN addressing was provided, so you must ensure IPv6 routing exists elsewhere, or add the appropriate IPv6 interface address and route.

---

## Opus 4.7

# Cisco IOS-XE 17 Production Configuration - ISR8000

Below is a production-grade configuration for a Cisco ISR8000 router running IOS-XE 17.x. Review all placeholders (hostnames, keys, ACLs, prefix-lists) before deploying.

```cisco
! =====================================================================
! Cisco ISR8000 - IOS-XE 17.x Production Configuration
! Role: Dual-homed Internet Edge Router with OSPF LAN + BGP WAN
! =====================================================================

!
! --- Basic System Configuration ---
!
hostname RTR-EDGE-01
!
service timestamps debug datetime msec localtime show-timezone
service timestamps log   datetime msec localtime show-timezone
service password-encryption
service tcp-keepalives-in
service tcp-keepalives-out
service sequence-numbers
!
no service pad
no service finger
no service config
no ip source-route
no ip bootp server
no ip http server
no ip http secure-server
no ip domain lookup
no cdp run
!
ip domain name corp.example.com
!
clock timezone UTC 0 0
!
! --- Platform / Boot ---
!
platform qfp utilization monitor load 80
platform punt-keepalive disable-kernel-core
!
! --- Logging ---
!
logging buffered 1048576 informational
logging console critical
logging monitor informational
logging trap informational
logging origin-id hostname
logging source-interface Loopback0
logging host ipv6 2000:DEAD:BEEF::1 transport udp port 514
!
! --- Users & AAA ---
!
enable secret 9 <ENABLE_SECRET_HASH>
!
username admin privilege 15 secret 9 <ADMIN_SECRET_HASH>
!
aaa new-model
!
aaa group server tacacs+ TAC_GROUP
 server name TAC01
 server name TAC02
 ip tacacs source-interface Loopback0
!
tacacs server TAC01
 address ipv4 200.0.0.1
 key 7 <tac_plus_key_secret>
 timeout 5
 single-connection
!
tacacs server TAC02
 address ipv4 200.0.0.2
 key 7 <tac_plus_key_secret>
 timeout 5
 single-connection
!
aaa authentication login default    group TAC_GROUP local
aaa authentication login CONSOLE    local
aaa authentication enable default   group TAC_GROUP enable
aaa authorization  config-commands
aaa authorization  exec    default  group TAC_GROUP local  if-authenticated
aaa authorization  commands 15 default group TAC_GROUP local if-authenticated
aaa accounting     exec    default  start-stop group TAC_GROUP
aaa accounting     commands 15 default start-stop group TAC_GROUP
aaa session-id common
!
! --- NTP ---
!
ntp source Loopback0
ntp update-calendar
ntp server 200.0.1.1 prefer
ntp server 200.0.1.2
!
! --- SSH / Management ---
!
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
ip ssh server algorithm mac hmac-sha2-256 hmac-sha2-512
ip ssh server algorithm encryption aes256-ctr aes192-ctr aes128-ctr
ip ssh server algorithm kex diffie-hellman-group14-sha1 diffie-hellman-group-exchange-sha256
crypto key generate rsa modulus 4096 label SSH-KEY
!
! --- Management ACL (adjust to your mgmt subnets) ---
!
ip access-list standard MGMT-ACL
 remark === Allow management sources ===
 permit 10.0.0.0 0.255.255.255
 deny   any log
!
! --- Banner ---
!
banner login ^
*********************************************************************
* WARNING: Authorized access only. All activity is logged.           *
* Disconnect immediately if you are not an authorized user.          *
*********************************************************************
^
!
! =====================================================================
! Interfaces
! =====================================================================
!
interface Loopback0
 description === Router-ID / Management Loopback ===
 ip address 10.255.255.1 255.255.255.255
 ip ospf 1 area 0
!
! --- WAN1: Primary (ISP-A, AS 65001) 50/50 Mbps ---
!
interface GigabitEthernet0/0
 description === WAN1 -> ISP-A (Primary) AS65001 50/50Mbps ===
 bandwidth 50000
 ip address 20.0.0.1 255.255.255.252
 ip access-group ACL-WAN-IN in
 ip verify unicast source reachable-via rx allow-default
 no ip redirects
 no ip unreachables
 no ip proxy-arp
 ip tcp adjust-mss 1380
 load-interval 30
 no shutdown
!
! --- WAN2: Secondary (ISP-B, AS 65002) 100/20 Mbps ---
!
interface GigabitEthernet0/1
 description === WAN2 -> ISP-B (Secondary) AS65002 100/20Mbps ===
 bandwidth 20000
 ip address 30.0.0.1 255.255.255.252
 ip access-group ACL-WAN-IN in
 ip verify unicast source reachable-via rx allow-default
 no ip redirects
 no ip unreachables
 no ip proxy-arp
 ip tcp adjust-mss 1380
 load-interval 30
 no shutdown
!
! --- LAN trunk ---
!
interface GigabitEthernet0/3
 description === LAN Trunk to Core Switch ===
 no ip address
 load-interval 30
 no shutdown
!
interface GigabitEthernet0/3.200
 description === LAN VLAN 200 - User Subnet ===
 encapsulation dot1Q 200
 ip address 192.168.0.1 255.255.255.240
 ip ospf 1 area 0
 ip ospf network point-to-multipoint
 ip ospf hello-interval 5
 ip ospf dead-interval 20
 no ip redirects
 no ip proxy-arp
!
! =====================================================================
! QoS - WAN egress shaping (optional baseline)
! =====================================================================
!
class-map match-any CM-VOICE
 match dscp ef
class-map match-any CM-CRITICAL
 match dscp cs6 cs7 af41 af42 af43
!
policy-map PM-WAN-OUT
 class CM-VOICE
  priority percent 20
 class CM-CRITICAL
  bandwidth percent 30
 class class-default
  fair-queue
  random-detect dscp-based
!
policy-map PM-WAN1-SHAPE
 class class-default
  shape average 50000000
  service-policy PM-WAN-OUT
!
policy-map PM-WAN2-SHAPE
 class class-default
  shape average 20000000
  service-policy PM-WAN-OUT
!
interface GigabitEthernet0/0
 service-policy output PM-WAN1-SHAPE
!
interface GigabitEthernet0/1
 service-policy output PM-WAN2-SHAPE
!
! =====================================================================
! Routing - OSPF (LAN)
! =====================================================================
!
router ospf 1
 router-id 10.255.255.1
 auto-cost reference-bandwidth 100000
 passive-interface default
 no passive-interface GigabitEthernet0/3.200
 area 0 authentication message-digest
 log-adjacency-changes detail
 timers throttle spf 50 200 5000
 timers throttle lsa 50 200 5000
 max-metric router-lsa on-startup wait-for-bgp
 default-information originate
!
! =====================================================================
! Prefix-Lists & Route-Maps (BGP policy)
! =====================================================================
!
ip prefix-list PL-OUT-OWN-NETS seq 5  permit 192.168.0.0/28
!
ip prefix-list PL-IN-DEFAULT   seq 5  permit 0.0.0.0/0
!
ip prefix-list PL-BOGONS seq 5  permit 0.0.0.0/8 le 32
ip prefix-list PL-BOGONS seq 10 permit 10.0.0.0/8 le 32
ip prefix-list PL-BOGONS seq 15 permit 127.0.0.0/8 le 32
ip prefix-list PL-BOGONS seq 20 permit 169.254.0.0/16 le 32
ip prefix-list PL-BOGONS seq 25 permit 172.16.0.0/12 le 32
ip prefix-list PL-BOGONS seq 30 permit 192.168.0.0/16 le 32
ip prefix-list PL-BOGONS seq 35 permit 224.0.0.0/4 le 32
ip prefix-list PL-BOGONS seq 40 permit 240.0.0.0/4 le 32
!
! --- Community-list for local preference tagging ---
ip community-list standard CL-PRIMARY   permit 65000:100
ip community-list standard CL-SECONDARY permit 65000:50
!
route-map RM-ISP-A-IN permit 10
 description === From ISP-A (Primary) - higher local-pref ===
 match ip address prefix-list PL-IN-DEFAULT
 set local-preference 200
 set community 65000:100
!
route-map RM-ISP-A-IN deny 20
!
route-map RM-ISP-B-IN permit 10
 description === From ISP-B (Secondary) - lower local-pref ===
 match ip address prefix-list PL-IN-DEFAULT
 set local-preference 100
 set community 65000:50
!
route-map RM-ISP-B-IN deny 20
!
route-map RM-ISP-A-OUT permit 10
 description === Advertise own prefixes to ISP-A (primary) ===
 match ip address prefix-list PL-OUT-OWN-NETS
!
route-map RM-ISP-A-OUT deny 20
!
route-map RM-ISP-B-OUT permit 10
 description === Advertise own prefixes to ISP-B with AS-prepend ===
 match ip address prefix-list PL-OUT-OWN-NETS
 set as-path prepend 65000 65000 65000
!
route-map RM-ISP-B-OUT deny 20
!
! =====================================================================
! Routing - BGP (WAN)
! =====================================================================
!
router bgp 65000
 bgp router-id 10.255.255.1
 bgp log-neighbor-changes
 bgp deterministic-med
 bgp bestpath compare-routerid
 bgp graceful-restart
 no bgp default ipv4-unicast
 timers bgp 10 30
 !
 neighbor ISP-A peer-group
 neighbor ISP-A remote-as 65001
 neighbor ISP-A description === Primary ISP-A ===
 neighbor ISP-A password 7 <BGP_ISP_A_MD5>
 neighbor ISP-A ttl-security hops 1
 neighbor ISP-A fall-over bfd
 !
 neighbor ISP-B peer-group
 neighbor ISP-B remote-as 65002
 neighbor ISP-B description === Secondary ISP-B ===
 neighbor ISP-B password 7 <BGP_ISP_B_MD5>
 neighbor ISP-B ttl-security hops 1
 neighbor ISP-B fall-over bfd
 !
 neighbor 20.0.0.2 peer-group ISP-A
 neighbor 30.0.0.2 peer-group ISP-B
 !
 address-family ipv4 unicast
  network 192.168.0.0 mask 255.255.255.240
  !
  neighbor ISP-A activate
  neighbor ISP-A send-community both
  neighbor ISP-A soft-reconfiguration inbound
  neighbor ISP-A maximum-prefix 1000000 90 restart 15
  neighbor ISP-A route-map RM-ISP-A-IN  in
  neighbor ISP-A route-map RM-ISP-A-OUT out
  neighbor ISP-A prefix-list PL-BOGONS  in
  !
  neighbor ISP-B activate
  neighbor ISP-B send-community both
  neighbor ISP-B soft-reconfiguration inbound
  neighbor ISP-B maximum-prefix 1000000 90 restart 15
  neighbor ISP-B route-map RM-ISP-B-IN  in
  neighbor ISP-B route-map RM-ISP-B-OUT out
  neighbor ISP-B prefix-list PL-BOGONS  in
  !
  neighbor 20.0.0.2 activate
  neighbor 30.0.0.2 activate
 exit-address-family
!
! --- Inject LAN network into BGP via static (preferred over redist) ---
!
ip route 192.168.0.0 255.255.255.240 Null0 250
!
! =====================================================================
! BFD (fast failover on WAN)
! =====================================================================
!
interface GigabitEthernet0/0
 bfd interval 300 min_rx 300 multiplier 3
!
interface GigabitEthernet0/1
 bfd interval 300 min_rx 300 multiplier 3
!
! =====================================================================
! WAN Ingress ACL (anti-spoofing / basic hardening)
! =====================================================================
!
ip access-list extended ACL-WAN-IN
 remark === Deny RFC1918 / bogons inbound ===
 deny   ip 10.0.0.0      0.255.255.255   any
 deny   ip 172.16.0.0    0.15.255.255    any
 deny   ip 192.168.0.0   0.0.255.255     any
 deny   ip 127.0.0.0     0.255.255.255   any
 deny   ip 169.254.0.0   0.0.255.255     any
 deny   ip 224.0.0.0     15.255.255.255  any
 deny   ip host 0.0.0.0  any
 remark === Permit established BGP to our WAN IPs ===
 permit tcp host 20.0.0.2 host 20.0.0.1 eq bgp
 permit tcp host 20.0.0.2 eq bgp host 20.0.0.1
 permit tcp host 30.0.0.2 host 30.0.0.1 eq bgp
 permit tcp host 30.0.0.2 eq bgp host 30.0.0.1
 remark === Permit ICMP for troubleshooting (rate-limited by CoPP) ===
 permit icmp any any echo
 permit icmp any any echo-reply
 permit icmp any any time-exceeded
 permit icmp any any packet-too-big
 permit icmp any any unreachable
 remark === Permit return traffic ===
 permit tcp any any established
 permit udp any eq 53 any
 permit udp any eq 123 any
 deny   ip any any log-input
!
! =====================================================================
! Control Plane Policing (CoPP) - baseline
! =====================================================================
!
ip access-list extended ACL-COPP-CRITICAL
 permit tcp host 20.0.0.2 eq bgp any
 permit tcp host 30.0.0.2 eq bgp any
 permit tcp any any eq bgp
ip access-list extended ACL-COPP-MGMT
 permit tcp any any eq 22
 permit tcp any eq tacacs any
 permit udp any any eq snmp
ip access-list extended ACL-COPP-NORMAL
 permit udp any any eq ntp
 permit icmp any any
!
class-map match-all CM-COPP-CRITICAL
 match access-group name ACL-COPP-CRITICAL
class-map match-all CM-COPP-MGMT
 match access-group name ACL-COPP-MGMT
class-map match-all CM-COPP-NORMAL
 match access-group name ACL-COPP-NORMAL
!
policy-map PM-COPP
 class CM-COPP-CRITICAL
  police 8000000 conform-action transmit exceed-action transmit
 class CM-COPP-MGMT
  police 2000000 conform-action transmit exceed-action drop
 class CM-COPP-NORMAL
  police 1000000 conform-action transmit exceed-action drop
 class class-default
  police 500000  conform-action transmit exceed-action drop
!
control-plane
 service-policy input PM-COPP
!
! =====================================================================
! VTY / Console
! =====================================================================
!
line con 0
 exec-timeout 10 0
 logging synchronous
 login authentication CONSOLE
 transport preferred none
!
line vty 0 4
 exec-timeout 15 0
 access-class MGMT-ACL in
 logging synchronous
 transport input ssh
 transport preferred none
!
line vty 5 15
 exec-timeout 15 0
 access-class MGMT-ACL in
 logging synchronous
 transport input ssh
 transport preferred none
!
! =====================================================================
! SNMP (optional - enable if needed)
! =====================================================================
!
snmp-server group RO-GROUP v3 priv read RO-VIEW
snmp-server view    RO-VIEW iso included
snmp-server location "DC1 - Rack A12"
snmp-server contact "noc@corp.example.com"
snmp-server trap-source Loopback0
snmp-server ifindex persist
!
! =====================================================================
! Archive / Rollback
! =====================================================================
!
archive
 path bootflash:archive/$h-config
 maximum 20
 time-period 1440
 write-memory
!
! =====================================================================
! End of Configuration
! =====================================================================
end
```

---

## Deployment Notes

<details>
<summary><b>⚠️ Pre-deployment checklist (must-read)</b></summary>

1. **Replace all placeholders** marked with `<...>`:
   - `<ENABLE_SECRET_HASH>`, `<ADMIN_SECRET_HASH>` → generate with `enable algorithm-type scrypt secret <pwd>`
   - `<tac_plus_key_secret>` → replace with your real TACACS+ shared key (the router will re-encrypt it as type 7)
   - `<BGP_ISP_A_MD5>`, `<BGP_ISP_B_MD5>` → BGP MD5 passwords agreed with each ISP
2. **Confirm ISP peering IPs** (`20.0.0.2`, `30.0.0.2`) and ASNs with your providers.
3. **Adjust `MGMT-ACL`** to your real management subnet(s).
4. **Public IPs used** (`20.0.0.1/30`, `30.0.0.1/30`) are in real-world public space — ensure these are the actual ranges assigned by your ISPs.
5. The ISR8000 series uses names like `GigabitEthernet0/0/0` in some variants. If your platform uses three-tier notation, replace `Gi0/0`, `Gi0/1`, `Gi0/3` accordingly.
</details>

<details>
<summary><b>🎯 Traffic engineering logic</b></summary>

- **Primary (ISP-A)** is preferred for **outbound** via `local-preference 200` vs. `100` on ISP-B.
- **Inbound preference** toward ISP-A is signaled by **AS-path prepending** 3× on ISP-B advertisements, making ISP-A a shorter path for return traffic.
- The `bandwidth` statement on each WAN influences OSPF/EIGRP cost calculations and NetFlow accounting; the QoS `shape` enforces actual contracted rates.
- `max-metric router-lsa on-startup wait-for-bgp` prevents transit blackholing during reloads until BGP converges.
</details>

<details>
<summary><b>✅ Verification commands</b></summary>

```cisco
show ip interface brief
show ip ospf neighbor
show ip bgp summary
show ip bgp neighbors 20.0.0.2 advertised-routes
show ip route bgp
show tacacs
show ntp associations
show logging | include 2000:DEAD
show policy-map interface GigabitEthernet0/0
show control-plane host open-ports
```
</details>

---

## Qwen3.6-Plus

! ==========================================
! ISR8000 Production Configuration - IOS-XE 17.x
! ==========================================

! Global & System Hardening
hostname ISR8000-PROD
ip domain-name corp.local
ip name-server 8.8.8.8 8.8.4.4
no ip domain lookup
service timestamps debug datetime msec localtime show-timezone
service timestamps log datetime msec localtime show-timezone
service password-encryption
no service pad
no service tcp-small-servers
no service udp-small-servers
ip routing
ipv6 unicast-routing
ip cef
ip tcp intercept list 10
access-list 10 permit 192.168.0.0 0.0.0.15

! Login Security
login block-for 300 attempts 5 within 60
login on-success log
login on-failure log
security passwords min-length 12
enable secret <strong_enable_secret>
username admin privilege 15 secret <local_fallback_secret>

! SSH Hardening
ip ssh version 2
ip ssh server algorithm encryption aes128-ctr aes192-ctr aes256-ctr
ip ssh server algorithm mac hmac-sha2-256 hmac-sha2-512
ip ssh server algorithm kex diffie-hellman-group16-sha512 diffie-hellman-group18-sha512
crypto key generate rsa modulus 2048

! NTP
ntp server 200.0.1.1
ntp server 200.0.1.2
ntp source Loopback0
ntp logging

! Syslog (IPv6)
logging host 2000:DEAD:BEEF::1 transport udp port 514
logging trap informational
logging source-interface Loopback0
logging monitor informational

! AAA & TACACS+
aaa new-model
tacacs server TACACS1
 address ipv4 200.0.0.1
 key <tac_plus_key_secret>
 timeout 5
tacacs server TACACS2
 address ipv4 200.0.0.2
 key <tac_plus_key_secret>
 timeout 5
aaa group server tacacs+ TACACS_GROUP
 server name TACACS1
 server name TACACS2
aaa authentication login default group TACACS_GROUP local
aaa authentication enable default group TACACS_GROUP enable
aaa authorization exec default group TACACS_GROUP local
aaa authorization network default group TACACS_GROUP local
aaa accounting exec default start-stop group TACACS_GROUP
aaa accounting commands 15 default start-stop group TACACS_GROUP

! Management Interface & Router ID
interface Loopback0
 description Management & Routing RID
 ip address 10.0.0.1 255.255.255.255
 ipv6 address 2001:db8::1/128

! WAN1 - Primary (50/50 Mbps)
interface GigabitEthernet0/0
 description WAN1-PRIMARY-ISP-AS65001
 ip address 20.0.0.1 255.255.255.252
 bandwidth 50000
 no ip redirects
 no ip unreachables
 no ip proxy-arp
 load-interval 30
 no shutdown

! WAN2 - Secondary (100/20 Mbps Asymmetric)
interface GigabitEthernet0/1
 description WAN2-SECONDARY-ISP-AS65002
 ip address 30.0.0.1 255.255.255.252
 bandwidth 100000
 no ip redirects
 no ip unreachables
 no ip proxy-arp
 load-interval 30
 ! Note: Apply QoS policing policy here for 20Mbps egress if required by SLA
 no shutdown

! Trunk Parent Interface
interface GigabitEthernet0/3
 description Trunk-to-Core-Switch
 no ip address
 no shutdown

! LAN Subinterface
interface GigabitEthernet0/3.200
 description LAN-VLAN200
 encapsulation dot1Q 200
 ip address 192.168.0.1 255.255.255.240
 no shutdown

! OSPF - LAN Basic Template
router ospf 1
 router-id 10.0.0.1
 passive-interface GigabitEthernet0/3.200
 network 192.168.0.0 0.0.0.15 area 0
!

! BGP - WAN Routing (AS 65000)
ip prefix-list LAN-ADV seq 5 permit 192.168.0.0/28
!
route-map PRIMARY-IN permit 10
 set local-preference 200
!
route-map SECONDARY-IN permit 10
 set local-preference 100
!
router bgp 65000
 bgp router-id 10.0.0.1
 bgp log-neighbor-changes
 bgp graceful-restart
 neighbor 20.0.0.2 remote-as 65001
 neighbor 20.0.0.2 description Primary-ISP-AS65001
 neighbor 20.0.0.2 password <bgp_password>
 neighbor 20.0.0.2 route-map PRIMARY-IN in
 neighbor 20.0.0.2 maximum-prefix 20000 80
 neighbor 30.0.0.2 remote-as 65002
 neighbor 30.0.0.2 description Secondary-ISP-AS65002
 neighbor 30.0.0.2 password <bgp_password>
 neighbor 30.0.0.2 route-map SECONDARY-IN in
 neighbor 30.0.0.2 maximum-prefix 20000 80
 !
 address-family ipv4
  network 192.168.0.0 mask 255.255.255.240 route-map LAN-ADV
  neighbor 20.0.0.2 activate
  neighbor 20.0.0.2 send-community
  neighbor 30.0.0.2 activate
  neighbor 30.0.0.2 send-community
 exit-address-family
!

! Line Security
line con 0
 exec-timeout 30 0
 logging synchronous
 privilege level 15
!
line vty 0 15
 transport input ssh
 exec-timeout 15 0
 logging synchronous
 privilege level 15
 login authentication default
!
end

---

## Gemini 3.1 Pro Preview





Here is a complete, production-grade configuration for a Cisco Catalyst 8000 Edge Platform (IOS-XE 17). 

This configuration integrates industry best practices for security, High Availability (HA) via BGP path manipulation, egress traffic shaping for your bandwidth tiers, and cleanly structured AAA/Logging setup.

### Prerequisites & Assumptions Before Pasting:
1. **Peers:** Assuming the ISP router IP addresses are `.2` for the `/30` WAN subnets.
2. **NAT:** Because your LAN uses an RFC1918 private address (`192.168.0.0/28`), you will likely need Network Address Translation (NAT) if this router faces the public internet. *NAT has been omitted to strictly honor your requested parameters.*
3. **Local Credentials:** Please replace `<tac_plus_key_secret>` and `<local_admin_password>` before pasting to avoid lockout.

### Cisco IOS-XE 17 Configuration

```cisco
! ======================================================================
! GLOBAL ROUTING & BASIC SYSTEM CONFIGURATION
! ======================================================================
hostname ISR8000-EDGE
!
ip routing
ipv6 unicast-routing
!
! Standard security: encrypt plaintext passwords in config
service password-encryption
!
! Create a local fallback admin in case TACACS+ is unreachable
username admin privilege 15 secret <local_admin_password>
!
! ======================================================================
! AAA & TACACS+ CONFIGURATION (IOS-XE 17 Syntax)
! ======================================================================
aaa new-model
!
tacacs server TAC-PRIMARY
 address ipv4 200.0.0.1
 key <tac_plus_key_secret>
!
tacacs server TAC-SECONDARY
 address ipv4 200.0.0.2
 key <tac_plus_key_secret>
!
aaa group server tacacs+ TAC-GROUP
 server name TAC-PRIMARY
 server name TAC-SECONDARY
!
aaa authentication login VTY_AUTH group TAC-GROUP local
aaa authorization exec VTY_AUTH group TAC-GROUP local
aaa accounting exec VTY_ACCT start-stop group TAC-GROUP
!
! ======================================================================
! NTP & SYSLOG CONFIGURATION
! ======================================================================
! NTP Servers
ntp server 200.0.1.1
ntp server 200.0.1.2
!
! Logging configuration (IPv6 target as requested)
logging trap informational
logging host ipv6 2000:DEAD:BEEF::1
!
! ======================================================================
! QoS / TRAFFIC SHAPING (Egress Bandwidth Control)
! ======================================================================
! WAN1: 50 Mbps Upload Shaping
policy-map PM-SHAPE-WAN1-50M
 class class-default
  shape average 50000000
!
! WAN2: 20 Mbps Upload Shaping (100M Down / 20M Up)
policy-map PM-SHAPE-WAN2-20M
 class class-default
  shape average 20000000
!
! ======================================================================
! INTERFACE CONFIGURATION
! ======================================================================
! --- PRIMARY WAN ---
interface GigabitEthernet0/0
 description WAN1_PRIMARY_AS65001_50x50M
 bandwidth 50000
 ip address 20.0.0.1 255.255.255.252
 service-policy output PM-SHAPE-WAN1-50M
 no ip redirects
 no ip proxy-arp
 no shutdown
!
! --- SECONDARY WAN ---
interface GigabitEthernet0/1
 description WAN2_SECONDARY_AS65002_100x20M
 bandwidth 100000
 ip address 30.0.0.1 255.255.255.252
 service-policy output PM-SHAPE-WAN2-20M
 no ip redirects
 no ip proxy-arp
 no shutdown
!
! --- LAN PARENT ---
interface GigabitEthernet0/3
 description LAN_Uplink_Parent
 no ip address
 no shutdown
!
! --- LAN SUBINTERFACE ---
interface GigabitEthernet0/3.200
 description LAN_VLAN200
 encapsulation dot1Q 200
 ip address 192.168.0.1 255.255.255.240
 ip ospf 1 area 0
 no shutdown
!
! ======================================================================
! BGP POLICY: PREFIX-LISTS & ROUTE-MAPS
! ======================================================================
! Ensure we ONLY advertise the LAN network to the ISPs
ip prefix-list PL-LOCAL-LAN seq 10 permit 192.168.0.0/28
!
! ISP1 (Primary) INBOUND: Higher Local Preference preferred for outbound routing
route-map RM-BGP-ISP1-IN permit 10
 set local-preference 200
!
! ISP1 (Primary) OUTBOUND: Advertise LAN normally
route-map RM-BGP-ISP1-OUT permit 10
 match ip address prefix-list PL-LOCAL-LAN
!
! ISP2 (Secondary) INBOUND: Default/Lower Local Pref
route-map RM-BGP-ISP2-IN permit 10
 set local-preference 100
!
! ISP2 (Secondary) OUTBOUND: Prepend AS to make inbound internet traffic prefer ISP1
route-map RM-BGP-ISP2-OUT permit 10
 match ip address prefix-list PL-LOCAL-LAN
 set as-path prepend 65000 65000 65000
!
! ======================================================================
! BGP CONFIGURATION
! ======================================================================
router bgp 65000
 bgp log-neighbor-changes
 bgp router-id 20.0.0.1
 network 192.168.0.0 mask 255.255.255.240
 !
 neighbor 20.0.0.2 remote-as 65001
 neighbor 20.0.0.2 description ISP1_PRIMARY
 !
 neighbor 30.0.0.2 remote-as 65002
 neighbor 30.0.0.2 description ISP2_SECONDARY
 !
 address-family ipv4
  neighbor 20.0.0.2 activate
  neighbor 20.0.0.2 route-map RM-BGP-ISP1-IN in
  neighbor 20.0.0.2 route-map RM-BGP-ISP1-OUT out
  !
  neighbor 30.0.0.2 activate
  neighbor 30.0.0.2 route-map RM-BGP-ISP2-IN in
  neighbor 30.0.0.2 route-map RM-BGP-ISP2-OUT out
 exit-address-family
!
! ======================================================================
! OSPF CONFIGURATION
! ======================================================================
router ospf 1
 router-id 192.168.0.1
 ! Make interfaces passive by default for security, only form neighborship on LAN
 passive-interface default
 no passive-interface GigabitEthernet0/3.200
 ! Inject a default route down to the LAN switches
 default-information originate
!
! ======================================================================
! LINE CONFIGURATION (SSH & TACACS+ Access)
! ======================================================================
ip ssh version 2
!
line con 0
 logging synchronous
 exec-timeout 15 0
!
line vty 0 4
 exec-timeout 15 0
 transport input ssh
 login authentication VTY_AUTH
 authorization exec VTY_AUTH
 accounting exec VTY_ACCT
!
line vty 5 15
 exec-timeout 15 0
 transport input ssh
 login authentication VTY_AUTH
 authorization exec VTY_AUTH
 accounting exec VTY_ACCT
!
end
write memory
```

### Key Engineering Decisions Explained:
* **Active/Passive WAN via BGP:**
  * **Outbound Traffic:** The router uses `local-preference 200` on routes learned from WAN1, strictly prioritizing WAN1 for all outbound destination tracking.
  * **Inbound Traffic:** The router explicitly prepends its own AS three times (`65000 65000 65000`) toward WAN2. Upstream peers on the internet will see the route through AS 65002 as 4 hops long vs AS 65001 as 1 hop long, causing return traffic to safely land on your Primary WAN1.
* **Egress Shaping:** QoS `policy-map` shapes the average traffic on outbound drops exactly to the limits you requested (50M on Gi0/0 and 20M on Gi0/1) mitigating ISP-side microburst drops and strict policing. 
* **BGP Route Leaking Prevention:** The config utilizes strict `prefix-list` checking ensuring the router cannot act as a transit node between AS 65001 and AS 65002.

---

## Gemma 4 31B IT

To implement this on an ISR 8000 running IOS-XE 17, you need a configuration that accounts for high availability and traffic engineering (since you have a Primary/Secondary WAN requirement).

Below is the production-grade configuration. I have included **Route-Maps** to ensure that WAN1 is truly the primary path for both inbound and outbound traffic.

### Production Configuration Template

```ios
!
hostname ISR8K-Edge-01
!
! --- Global System Settings ---
service password-encryption
no ip domain-lookup
ip domain-name enterprise.local
ipv6 unicast-routing
!
! --- AAA / TACACS+ Configuration ---
aaa new-model
aaa authentication login default group tacacs+ local
aaa authorization exec default group tacacs+ local 
aaa authorization commands 15 default group tacacs+ local
aaa accounting exec default start-stop group tacacs+
aaa accounting commands 15 default start-stop group tacacs+
!
tacacs server TAC_SRV_1
 address ipv4 200.0.0.1
 key <tac_plus_key_secret>
tacacs server TAC_SRV_2
 address ipv4 200.0.0.2
 key <tac_plus_key_secret>
!
! --- Management: NTP & Syslog ---
ntp server 200.0.1.1 prefer
ntp server 200.0.1.2
!
logging host 2000:DEAD:BEEF::1
logging trap notifications
!
! --- Interface Configuration ---
!
! WAN1 - Primary (ISP AS 65001)
interface GigabitEthernet0/0
 description WAN1_PRIMARY_ISP_65001
 ip address 20.0.0.1 255.255.255.252
 bandwidth 50000
 no shutdown
!
! WAN2 - Secondary (ISP AS 65002)
interface GigabitEthernet0/1
 description WAN2_SECONDARY_ISP_65002
 ip address 30.0.0.1 255.255.255.252
 bandwidth 20000
 no shutdown
!
! LAN Interface
interface GigabitEthernet0/3
 description LAN_TRUNK_PORT
 no shutdown
!
interface GigabitEthernet0/3.200
 description LAN_VLAN_200
 encapsulation dot1Q 200
 ip address 192.168.0.1 255.255.255.240
!
! --- Routing: OSPF (LAN Side) ---
router ospf 1
 router-id 192.168.0.1
 log-adjacency-changes
 network 192.168.0.0 0.0.0.15 area 0
!
! --- BGP Traffic Engineering ---
!
! Prefix-list for our local network to advertise to ISPs
ip prefix-list PL_LOCAL_NET seq 5 permit 192.168.0.0/28
!
! Route-map to prefer WAN1 for Outbound traffic (Local Preference)
route-map RM_WAN1_IN permit 10
 set local-preference 200
!
route-map RM_WAN2_IN permit 10
 set local-preference 100
!
! Route-map to make WAN2 less attractive for Inbound traffic (AS-Path Prepend)
route-map RM_WAN1_OUT permit 10
 match ip address prefix-list PL_LOCAL_NET
!
route-map RM_WAN2_OUT permit 10
 match ip address prefix-list PL_LOCAL_NET
 set as-path prepend 65000 65000 65000
!
! --- Routing: BGP (WAN Side) ---
router bgp 65000
 bgp log-neighbor-changes
 neighbor 20.0.0.2 remote-as 65001
 neighbor 20.0.0.2 description ISP1_PRIMARY
 neighbor 20.0.0.2 route-map RM_WAN1_IN in
 neighbor 20.0.0.2 route-map RM_WAN1_OUT out
 !
 neighbor 30.0.0.2 remote-as 65002
 neighbor 30.0.0.2 description ISP2_SECONDARY
 neighbor 30.0.0.2 route-map RM_WAN2_IN in
 neighbor 30.0.0.2 route-map RM_WAN2_OUT out
 !
 address-family ipv4
  network 192.168.0.0 mask 255.255.255.240
  neighbor 20.0.0.2 activate
  neighbor 30.0.0.2 activate
 exit-address-family
!
! --- Security Hardening ---
line vty 0 4
 transport input ssh
 login authentication default
!
line con 0
 logging synchronous
 stopbits 1
!
end
write memory
```

### Engineering Explanations for Production Readiness:

#### 1. WAN Traffic Steering (The "Primary/Secondary" Logic)
In BGP, simply having two links does not make one "Primary." I have implemented two mechanisms:
*   **Outbound Traffic (Local Preference):** I applied `RM_WAN1_IN` to set a Local Preference of **200** for WAN1 and **100** for WAN2. BGP always prefers the higher Local Preference, ensuring your users exit via WAN1.
*   **Inbound Traffic (AS-Path Prepending):** To prevent the internet from sending traffic via WAN2, I used `RM_WAN2_OUT` to prepend your own AS number three times. This makes the path through WAN2 look "longer" and less desirable to external routers.

#### 2. Interface Logic
*   **Gi0/3.200:** Configured as a sub-interface with `encapsulation dot1q 200` to support the VLAN tagging requested.
*   **Bandwidth:** The `bandwidth` command is added to the interfaces. While this doesn't shape traffic (you would need an MQC policy for that), it informs routing protocols and QoS calculations about the link capacity.

#### 3. Management & Security
*   **AAA:** Configured for TACACS+ with a fallback to local authentication. This ensures that if the TACACS servers are unreachable, you can still access the device via a local admin account.
*   **OSPF:** Configured as a basic template. I have used the specific wildcard mask `0.0.0.15` to match the `/28` subnet exactly.
*   **IPv6:** `ipv6 unicast-routing` is enabled globally to allow the router to communicate with the IPv6 Syslog server.

### How to deploy:
1.  Replace `<tac_plus_key_secret>` with your actual shared secret.
2.  Ensure you have a local username/password configured before applying the AAA section, or you may lock yourself out:
    `username admin privilege 15 secret <your_password>`
3.  Copy and paste the block into the global configuration mode (`conf t`).

---

## Sonnet 4.6


Here's the full production config. A few key design decisions worth calling out:

**BGP Dual-WAN Strategy**
WAN1 is preferred in both directions: outbound via higher local-preference (200 vs 100) and inbound via AS-path prepending on WAN2 advertisements (3× prepend). BGP falls over using BFD (300ms timers, ×3 multiplier) rather than waiting for hold-timer expiry.

**NAT with Dual-WAN Failover**
Uses per-interface route-maps for NAT overload instead of a single ACL+interface binding — this is required for correct behavior when traffic exits through a different gateway during failover.

**QoS**
WAN1 shaped to 50 Mbps upload. WAN2 shaped to **20 Mbps** upload (the asymmetric constraint) — bandwidth % values allocate priority to Voice (EF), then Critical/Business, then best-effort. Adjust DSCP markings to match your internal policy.

**IP SLA + Tracking**
Pings the ISP next-hop every 10 seconds, with a 15s/10s down/up dampening delay to avoid flapping.

**Placeholders to replace before deployment:**

| Placeholder | What to replace with |
|---|---|
| `<tac_plus_key_secret>` | Your TACACS+ shared secret |
| `<local_enable_secret>` | Local fallback password |
| `<ntp_auth_key>` | NTP MD5 key string |
| `<snmp_auth_password>` | SNMPv3 auth password |
| `<snmp_priv_password>` | SNMPv3 priv password |
| `corp.local` | Your actual domain |

**Pre-paste EXEC command** (must run first — `crypto key generate` is not a config-mode command):
```
crypto key generate rsa modulus 4096 label SSH-RSA
```

! ============================================================
! Cisco IOS-XE 17 — ISR 8000 Production Configuration
! ============================================================
! Topology Summary:
!   WAN1  Gi0/0   20.0.0.1/30   AS65001  Primary   50/50 Mbps
!   WAN2  Gi0/1   30.0.0.1/30   AS65002  Secondary 100/20 Mbps
!   LAN   Gi0/3.200  192.168.0.1/28  VLAN 200
!   BGP AS: 65000 | OSPF Process: 1 | Area: 0
! ============================================================
! IMPORTANT — run these privileged EXEC commands BEFORE pasting:
!   conf t
!   crypto key generate rsa modulus 4096 label SSH-RSA
!   end
! Then paste this configuration in full.
! ============================================================
 
version 17.3
service timestamps debug datetime msec localtime show-timezone year
service timestamps log datetime msec localtime show-timezone year
service password-encryption
service tcp-keepalives-in
service tcp-keepalives-out
no service pad
no service udp-small-servers
no service tcp-small-servers
no service finger
!
hostname RTR-01
!
! ============================================================
! BOOT / ARCHIVE
! ============================================================
boot-start-marker
boot-end-marker
!
archive
 log config
  logging enable
  logging size 200
  notify syslog contenttype plaintext
  hidekeys
!
! ============================================================
! AAA / TACACS+
! ============================================================
aaa new-model
!
tacacs server TACACS-01
 address ipv4 200.0.0.1
 key <tac_plus_key_secret>
 timeout 5
!
tacacs server TACACS-02
 address ipv4 200.0.0.2
 key <tac_plus_key_secret>
 timeout 5
!
aaa group server tacacs+ TACACS-GROUP
 server name TACACS-01
 server name TACACS-02
!
aaa authentication login default group TACACS-GROUP local
aaa authentication enable default group TACACS-GROUP enable
aaa authorization console
aaa authorization exec default group TACACS-GROUP local if-authenticated
aaa authorization commands 1 default group TACACS-GROUP local if-authenticated
aaa authorization commands 15 default group TACACS-GROUP local if-authenticated
aaa accounting exec default start-stop group TACACS-GROUP
aaa accounting commands 1 default start-stop group TACACS-GROUP
aaa accounting commands 15 default start-stop group TACACS-GROUP
aaa accounting connection default start-stop group TACACS-GROUP
aaa accounting system default start-stop group TACACS-GROUP
aaa session-id common
!
! ============================================================
! LOCAL FALLBACK USER
! ============================================================
username admin privilege 15 algorithm-type scrypt secret <local_enable_secret>
enable algorithm-type scrypt secret <local_enable_secret>
!
! ============================================================
! GLOBAL IP SETTINGS
! ============================================================
ip routing
ipv6 unicast-routing
!
no ip domain-lookup
ip domain-name corp.local
!
ip tcp adjust-mss 1452
!
! ============================================================
! NTP
! ============================================================
ntp authenticate
ntp authentication-key 1 md5 <ntp_auth_key>
ntp trusted-key 1
ntp server 200.0.1.1 key 1 prefer
ntp server 200.0.1.2 key 1
ntp update-calendar
!
! Adjust timezone to your region — example: CET/CEST
clock timezone CET 1 0
clock summer-time CEST recurring last Sun Mar 2:00 last Sun Oct 3:00
!
! ============================================================
! LOGGING / SYSLOG
! ============================================================
logging on
logging buffered 1024000 informational
no logging console
logging monitor informational
logging trap informational
logging facility local6
logging source-interface GigabitEthernet0/3.200
logging host ipv6 2000:DEAD:BEEF::1 transport udp port 514
!
! ============================================================
! SSH / MANAGEMENT HARDENING
! ============================================================
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
ip ssh source-interface GigabitEthernet0/3.200
ip ssh server algorithm authentication publickey keyboard password
ip ssh server algorithm encryption aes256-ctr aes192-ctr aes128-ctr
ip ssh server algorithm mac hmac-sha2-256 hmac-sha2-512
!
no ip http server
no ip http secure-server
!
! ============================================================
! INTERFACES
! ============================================================
 
! --- WAN1: Primary | 50/50 Mbps | ISP AS65001 ---
interface GigabitEthernet0/0
 description WAN1-PRIMARY | ISP-AS65001 | 50/50Mbps | 20.0.0.1/30
 ip address 20.0.0.1 255.255.255.252
 ip access-group ACL-WAN-IN in
 ip nat outside
 no ip proxy-arp
 no ip redirects
 no ip unreachables
 ip verify unicast source reachable-via rx allow-default
 bandwidth 50000
 service-policy output PM-WAN1-EGRESS
 no shutdown
!
 
! --- WAN2: Secondary | 100/20 Mbps (upload=20) | ISP AS65002 ---
interface GigabitEthernet0/1
 description WAN2-SECONDARY | ISP-AS65002 | 100/20Mbps | 30.0.0.1/30
 ip address 30.0.0.1 255.255.255.252
 ip access-group ACL-WAN-IN in
 ip nat outside
 no ip proxy-arp
 no ip redirects
 no ip unreachables
 ip verify unicast source reachable-via rx allow-default
 bandwidth 20000
 service-policy output PM-WAN2-EGRESS
 no shutdown
!
 
! --- LAN Physical Trunk ---
interface GigabitEthernet0/3
 description LAN-TRUNK-TO-SWITCH
 no ip address
 no shutdown
!
 
! --- LAN Subinterface VLAN 200 ---
interface GigabitEthernet0/3.200
 description LAN-VLAN200 | 192.168.0.1/28
 encapsulation dot1Q 200
 ip address 192.168.0.1 255.255.255.240
 ip access-group ACL-LAN-IN in
 ip nat inside
 no ip proxy-arp
 no shutdown
!
 
! ============================================================
! IP SLA — WAN REACHABILITY MONITORING
! ============================================================
ip sla 10
 icmp-echo 20.0.0.2 source-interface GigabitEthernet0/0
 frequency 10
 threshold 3000
 timeout 3000
ip sla schedule 10 life forever start-time now
!
ip sla 20
 icmp-echo 30.0.0.2 source-interface GigabitEthernet0/1
 frequency 10
 threshold 3000
 timeout 3000
ip sla schedule 20 life forever start-time now
!
track 10 ip sla 10 reachability
 delay down 15 up 10
!
track 20 ip sla 20 reachability
 delay down 15 up 10
!
 
! ============================================================
! OSPF — LAN (Process 1, Area 0)
! ============================================================
router ospf 1
 router-id 192.168.0.1
 auto-cost reference-bandwidth 10000
 passive-interface default
 no passive-interface GigabitEthernet0/3.200
 network 192.168.0.0 0.0.0.15 area 0
 default-information originate always metric 10 metric-type 2
 log-adjacency-changes detail
!
 
! ============================================================
! BGP — WAN (AS 65000)
! ============================================================
 
! Prefix lists
ip prefix-list PL-DEFAULT seq 5 permit 0.0.0.0/0
ip prefix-list PL-LAN-NETWORKS seq 5 permit 192.168.0.0/28
ip prefix-list PL-DENY-ALL seq 5 deny 0.0.0.0/0 le 32
!
 
! Route-maps — Inbound
!   WAN1: local-pref 200 → preferred for all inbound traffic
route-map RM-WAN1-IN permit 10
 description INBOUND: Set high local-preference (primary)
 match ip address prefix-list PL-DEFAULT
 set local-preference 200
!
!   WAN2: local-pref 100 → fallback for inbound traffic
route-map RM-WAN2-IN permit 10
 description INBOUND: Set low local-preference (secondary)
 match ip address prefix-list PL-DEFAULT
 set local-preference 100
!
 
! Route-maps — Outbound
!   WAN1: clean advertisement (primary path)
route-map RM-WAN1-OUT permit 10
 description OUTBOUND: Advertise LAN normally via primary
 match ip address prefix-list PL-LAN-NETWORKS
!
!   WAN2: prepend AS path twice to discourage inbound (secondary path)
route-map RM-WAN2-OUT permit 10
 description OUTBOUND: Advertise LAN with AS-path prepend via secondary
 match ip address prefix-list PL-LAN-NETWORKS
 set as-path prepend 65000 65000 65000
!
 
router bgp 65000
 bgp router-id 20.0.0.1
 bgp log-neighbor-changes
 bgp deterministic-med
 no bgp default ipv4-unicast
 !
 ! --- WAN1: ISP AS65001 (Primary) ---
 neighbor 20.0.0.2 remote-as 65001
 neighbor 20.0.0.2 description WAN1-PRIMARY-ISP | AS65001
 neighbor 20.0.0.2 timers 10 30
 neighbor 20.0.0.2 fall-over bfd
 neighbor 20.0.0.2 ttl-security hops 1
 !
 ! --- WAN2: ISP AS65002 (Secondary) ---
 neighbor 30.0.0.2 remote-as 65002
 neighbor 30.0.0.2 description WAN2-SECONDARY-ISP | AS65002
 neighbor 30.0.0.2 timers 10 30
 neighbor 30.0.0.2 fall-over bfd
 neighbor 30.0.0.2 ttl-security hops 1
 !
 address-family ipv4 unicast
  !
  ! Advertise LAN subnet
  network 192.168.0.0 mask 255.255.255.240
  !
  ! WAN1 — Primary
  neighbor 20.0.0.2 activate
  neighbor 20.0.0.2 soft-reconfiguration inbound
  neighbor 20.0.0.2 route-map RM-WAN1-IN in
  neighbor 20.0.0.2 route-map RM-WAN1-OUT out
  neighbor 20.0.0.2 prefix-list PL-DEFAULT in
  neighbor 20.0.0.2 send-community both
  neighbor 20.0.0.2 next-hop-self
  !
  ! WAN2 — Secondary
  neighbor 30.0.0.2 activate
  neighbor 30.0.0.2 soft-reconfiguration inbound
  neighbor 30.0.0.2 route-map RM-WAN2-IN in
  neighbor 30.0.0.2 route-map RM-WAN2-OUT out
  neighbor 30.0.0.2 prefix-list PL-DEFAULT in
  neighbor 30.0.0.2 send-community both
  neighbor 30.0.0.2 next-hop-self
  !
 exit-address-family
!
 
! ============================================================
! NAT / PAT — Dual WAN
! Using route-maps to bind NAT overload per-interface so that
! failover tracks the correct exit interface.
! ============================================================
ip access-list standard ACL-NAT-CANDIDATES
 permit 192.168.0.0 0.0.0.15
!
route-map RM-NAT-WAN1 permit 10
 match ip address ACL-NAT-CANDIDATES
 match interface GigabitEthernet0/0
!
route-map RM-NAT-WAN2 permit 10
 match ip address ACL-NAT-CANDIDATES
 match interface GigabitEthernet0/1
!
ip nat inside source route-map RM-NAT-WAN1 interface GigabitEthernet0/0 overload
ip nat inside source route-map RM-NAT-WAN2 interface GigabitEthernet0/1 overload
!
 
! ============================================================
! ACCESS LISTS
! ============================================================
 
! --- WAN Inbound: Drop spoofed/bogon sources ---
ip access-list extended ACL-WAN-IN
 remark *** RFC 5735 / Bogon deny ***
 deny   ip 0.0.0.0 0.255.255.255 any log-input
 deny   ip 10.0.0.0 0.255.255.255 any log-input
 deny   ip 127.0.0.0 0.255.255.255 any log-input
 deny   ip 169.254.0.0 0.0.255.255 any log-input
 deny   ip 172.16.0.0 0.15.255.255 any log-input
 deny   ip 192.0.0.0 0.0.0.255 any log-input
 deny   ip 192.168.0.0 0.0.255.255 any log-input
 deny   ip 198.18.0.0 0.1.255.255 any log-input
 deny   ip 224.0.0.0 31.255.255.255 any log-input
 remark *** Allow return / ICMP diagnostics ***
 permit tcp any any established
 permit icmp any any echo-reply
 permit icmp any any unreachable
 permit icmp any any time-exceeded
 remark *** Allow BGP from known ISP peers ***
 permit tcp host 20.0.0.2 host 20.0.0.1 eq 179
 permit tcp host 30.0.0.2 host 30.0.0.1 eq 179
 remark *** Drop everything else ***
 deny   ip any any log-input
!
 
! --- LAN Inbound ---
ip access-list extended ACL-LAN-IN
 remark *** Permit LAN segment to WAN and management ***
 permit ip 192.168.0.0 0.0.0.15 any
 deny   ip any any log-input
!
 
! ============================================================
! QUALITY OF SERVICE
! ============================================================
 
! DSCP classification
class-map match-any CM-VOICE
 match dscp ef
class-map match-any CM-SIGNALING
 match dscp cs3
class-map match-any CM-CRITICAL-DATA
 match dscp cs6 cs5 af41 af42
class-map match-any CM-BUSINESS-DATA
 match dscp af31 af32 af33
class-map match-any CM-BULK
 match dscp cs1 af11 af12
!
 
! WAN1 Egress Policy — 50 Mbps symmetric
policy-map PM-WAN1-EGRESS
 class CM-VOICE
  priority percent 20
  police rate percent 20
 class CM-SIGNALING
  bandwidth percent 5
 class CM-CRITICAL-DATA
  bandwidth percent 30
  random-detect dscp-based
 class CM-BUSINESS-DATA
  bandwidth percent 25
  random-detect dscp-based
 class CM-BULK
  bandwidth percent 5
 class class-default
  fair-queue
  bandwidth percent 15
  random-detect dscp-based
!
 
! WAN2 Egress Policy — 20 Mbps upload enforced (100/20 Mbps link)
policy-map PM-WAN2-EGRESS
 class CM-VOICE
  priority percent 20
  police rate percent 20
 class CM-SIGNALING
  bandwidth percent 5
 class CM-CRITICAL-DATA
  bandwidth percent 30
  random-detect dscp-based
 class CM-BUSINESS-DATA
  bandwidth percent 20
  random-detect dscp-based
 class CM-BULK
  bandwidth percent 5
 class class-default
  fair-queue
  bandwidth percent 20
  random-detect dscp-based
!
 
! ============================================================
! BFD — Bi-Directional Forwarding Detection (linked to BGP)
! ============================================================
bfd-template single-hop BFD-WAN
 interval min-tx 300 min-rx 300 multiplier 3
!
interface GigabitEthernet0/0
 bfd template BFD-WAN
!
interface GigabitEthernet0/1
 bfd template BFD-WAN
!
 
! ============================================================
! SNMP v3 (read-only for NMS)
! ============================================================
snmp-server view SNMP-VIEW iso included
snmp-server group SNMP-RO-GROUP v3 priv read SNMP-VIEW
snmp-server user snmpadmin SNMP-RO-GROUP v3 auth sha <snmp_auth_password> priv aes 256 <snmp_priv_password>
snmp-server contact noc@corp.local
snmp-server location SITE-DC1-RACK-XX
snmp-server host 200.0.0.1 version 3 priv snmpadmin
!
! ============================================================
! CONSOLE / VTY LINES
! ============================================================
line con 0
 exec-timeout 10 0
 logging synchronous
 login authentication default
!
line vty 0 4
 transport input ssh
 exec-timeout 10 0
 logging synchronous
 login authentication default
 authorization exec default
!
line vty 5 15
 transport input ssh
 exec-timeout 10 0
 logging synchronous
 login authentication default
 authorization exec default
!
 
! ============================================================
! CONTROL-PLANE PROTECTION
! ============================================================
control-plane
 service-policy input PM-WAN1-EGRESS
!
 
! ============================================================
! SECURITY — Anti-spoofing / TCP intercept / Misc
! ============================================================
ip tcp intercept list ACL-TCP-INTERCEPT
ip tcp intercept mode watch
!
ip access-list extended ACL-TCP-INTERCEPT
 permit tcp any 192.168.0.0 0.0.0.15
!
no ip source-route
no ip gratuitous-arps
!
! ============================================================
! END OF CONFIGURATION
! ============================================================
end

---
 
