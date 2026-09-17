# 🌐 MikroTik hAP ax³ — Dual-WAN RouterOS 7 Setup

Complete configuration and security documentation for a **MikroTik hAP ax³** with:

* ⭐ **ether1** → Starlink
* 📡 **ether4** → Microwave Internet Link
* 🖥️ **ether2 / ether3 / ether5** → LAN
* 🔄 Starlink → Primary WAN
* 🛟 Microwave Internet → Backup WAN
* 🏠 LAN → `192.168.88.0/24`
* 🌐 DNS → Cloudflare + Google
* 🔀 Dual-WAN NAT
* 🛡️ Firewall and management hardening
* 🕐 Kyiv time zone + NTP
* 📶 Wi-Fi monitoring
* 💾 Configuration backup and export
* 🔀 Future PCC 50/50 load balancing

---

# 🧾 Device Information

| Parameter      | Value                 |
| -------------- | --------------------- |
| Device         | MikroTik hAP ax³      |
| Board          | `C53UiG+5HPaxD2HPaxD` |
| RouterOS       | `7.12.2`              |
| Identity       | `MikroTik`            |
| Management MAC | `XX:XX:XX:XX:XX:89`   |
| IP             | `0.0.0.0`             |
| Board Port     | `bridgeLocal/ether2`  |
| User           | `****`              |
| Group          | `full`                |

### Hardware Information

```text
Model: hAP ax³

ID: C53UiG+5HPaxD2HPaxD
FCC ID: TV7C53-5AXD2AXD
IC: 7442A-C53AX

Ethernet:
E01: XX:XX:XX:XX:XX:88
E02: XX:XX:XX:XX:XX:89

Wireless:
W01: XX:XX:XX:XX:XX:8E

Serial:
[REDACTED]

Default user:
admin

Default password:
[REDACTED]

Wi-Fi key:
[REDACTED]
```

> 🔐 **Security:** Do not publish administrator passwords, Wi-Fi keys, private keys or other credentials in GitHub repositories.

---

# 🗺️ Network Topology

```text
                         INTERNET
                        /        \
                       /          \
                  STARLINK      MICROWAVE
                     │              │
                  ether1          ether4
                     │              │
                     ▼              ▼
               ┌────────────────────────┐
               │      MikroTik hAP ax³  │
               │                        │
               │       DUAL-WAN         │
               │                        │
               │  Starlink = Primary    │
               │  Microwave = Backup    │
               └───────────┬────────────┘
                           │
                    ┌──────┼──────┐
                    │      │      │
                  ether2  ether3  ether5
                    │      │      │
                    └──────┼──────┘
                           │
                          LAN
                    192.168.88.0/24
```

---

# 🔌 1. Port Assignment

| Port     | Purpose               |
| -------- | --------------------- |
| `ether1` | ⭐ Starlink            |
| `ether2` | 🖥️ LAN               |
| `ether3` | 🖥️ LAN               |
| `ether4` | 📡 Microwave Internet |
| `ether5` | 🖥️ LAN               |

Final layout:

```text
WAN:
ether1 → Starlink
ether4 → Microwave Internet

LAN:
ether2
ether3
ether5
```

> ℹ️ Starlink and the microwave link do **not** need to be connected during the initial LAN/security configuration. Their actual IP addresses and gateways should be learned from the providers after connection.

---

# 🔍 2. Check Interfaces

Open:

**WinBox → Terminal**

```routeros
/interface print
```

Check bridge membership:

```routeros
/interface bridge port print
```

The default LAN bridge is:

```text
bridgeLocal
```

Initially, Ethernet ports may be members of this bridge.

---

# 🚫 3. Remove WAN Ports from LAN Bridge

`ether1` and `ether4` must not remain in the LAN bridge.

First check:

```routeros
/interface bridge port print
```

Then remove the corresponding entries.

For example:

```routeros
/interface bridge port remove [find interface=ether1]
```

```routeros
/interface bridge port remove [find interface=ether4]
```

Using `find interface=...` is safer than relying on changing numeric item IDs.

Verify:

```routeros
/interface bridge port print
```

Expected LAN bridge membership:

```text
ether2
ether3
ether5
```

Final layout:

```text
ether1 → WAN
ether4 → WAN

ether2 → LAN
ether3 → LAN
ether5 → LAN
```

---

# 🏠 4. Configure LAN Address

The LAN gateway is:

```text
192.168.88.1/24
```

Add the address:

```routeros
/ip address add \
address=192.168.88.1/24 \
interface=bridgeLocal \
comment="LAN"
```

Verify:

```routeros
/ip address print
```

Expected:

```text
192.168.88.1/24    bridgeLocal
```

---

# 🧹 5. Remove the Default DHCP Client

Check existing DHCP clients:

```routeros
/ip dhcp-client print detail
```

If an old DHCP client is attached to `bridgeLocal`, remove it:

```routeros
/ip dhcp-client remove [find interface=bridgeLocal]
```

Verify:

```routeros
/ip dhcp-client print
```

The LAN bridge should not use a DHCP client.

---

# 🌐 6. Configure Dual-WAN DHCP Clients

## ⭐ Starlink — Primary WAN

```routeros
/ip dhcp-client add \
interface=ether1 \
add-default-route=yes \
default-route-distance=1 \
use-peer-dns=no \
comment="WAN1 - Starlink"
```

## 📡 Microwave — Backup WAN

```routeros
/ip dhcp-client add \
interface=ether4 \
add-default-route=yes \
default-route-distance=2 \
use-peer-dns=no \
comment="WAN2 - Microwave Internet"
```

Verify:

```routeros
/ip dhcp-client print detail
```

Before the WAN cables are connected, it is normal for the DHCP clients to be inactive/searching.

---

# 🏠 7. Configure LAN DHCP

Create the address pool:

```routeros
/ip pool add \
name=LAN-Pool \
ranges=192.168.88.10-192.168.88.254
```

Verify:

```routeros
/ip pool print
```

Expected:

```text
LAN-Pool    192.168.88.10-192.168.88.254
```

Create the DHCP server:

```routeros
/ip dhcp-server add \
name=LAN-DHCP \
interface=bridgeLocal \
address-pool=LAN-Pool \
lease-time=1d \
disabled=no
```

Configure the DHCP network:

```routeros
/ip dhcp-server network add \
address=192.168.88.0/24 \
gateway=192.168.88.1 \
dns-server=192.168.88.1
```

Verify:

```routeros
/ip dhcp-server print
```

```routeros
/ip dhcp-server network print
```

Expected:

```text
Network:  192.168.88.0/24
Gateway:  192.168.88.1
DNS:      192.168.88.1
```

---

# 🧠 8. DNS Configuration

Enable DNS forwarding for LAN clients:

```routeros
/ip dns set allow-remote-requests=yes
```

Configure external DNS servers:

```routeros
/ip dns set servers=1.1.1.1,8.8.8.8
```

Verify:

```routeros
/ip dns print
```

Expected:

```text
servers: 1.1.1.1,8.8.8.8
allow-remote-requests: yes
```

---

# 🛡️ 9. Interface Lists

Interface lists simplify firewall configuration.

Create:

```routeros
/interface list add name=LAN
```

```routeros
/interface list add name=WAN
```

Add LAN:

```routeros
/interface list member add \
list=LAN \
interface=bridgeLocal
```

Add WAN1:

```routeros
/interface list member add \
list=WAN \
interface=ether1
```

Add WAN2:

```routeros
/interface list member add \
list=WAN \
interface=ether4
```

Verify:

```routeros
/interface list member print
```

Expected:

```text
LAN → bridgeLocal

WAN → ether1
WAN → ether4
```

---

# 🔐 10. Management Services

Management access should be available only from the LAN.

Allowed:

```text
FTP    → 21
SSH    → 22
WinBox → 8291
```

Restricted to:

```text
192.168.88.0/24
```

Configure:

```routeros
/ip service set ftp address=192.168.88.0/24
```

```routeros
/ip service set ssh address=192.168.88.0/24
```

```routeros
/ip service set winbox address=192.168.88.0/24
```

Disable unused services:

```routeros
/ip service disable telnet
```

```routeros
/ip service disable www
```

```routeros
/ip service disable api
```

```routeros
/ip service disable api-ssl
```

Verify:

```routeros
/ip service print detail
```

---

# 🧱 11. Firewall — INPUT Chain

The `input` chain controls traffic destined for the MikroTik itself.

Desired behavior:

```text
LAN → MikroTik       ALLOW
WAN → MikroTik       DROP

Established          ALLOW
Related              ALLOW
Invalid              DROP
Everything else      DROP
```

## Established / Related

```routeros
/ip firewall filter add \
chain=input \
action=accept \
connection-state=established,related,untracked \
comment="INPUT - established, related"
```

## Invalid

```routeros
/ip firewall filter add \
chain=input \
action=drop \
connection-state=invalid \
comment="INPUT - drop invalid"
```

## Allow LAN

```routeros
/ip firewall filter add \
chain=input \
action=accept \
in-interface-list=LAN \
comment="INPUT - allow LAN"
```

## Log New WAN Connections

```routeros
/ip firewall filter add \
chain=input \
action=log \
log-prefix="FW INPUT DROP: " \
connection-state=new \
in-interface-list=WAN \
comment="LOG - blocked WAN input"
```

## Drop WAN

```routeros
/ip firewall filter add \
chain=input \
action=drop \
in-interface-list=WAN \
comment="INPUT - drop WAN"
```

## Default Drop

```routeros
/ip firewall filter add \
chain=input \
action=drop \
comment="INPUT - drop everything else"
```

---

# 🌐 12. Firewall — FORWARD Chain

The `forward` chain controls traffic passing through the router.

Desired behavior:

```text
LAN → WAN       ALLOW
WAN → LAN       DROP

Established     ALLOW
Related         ALLOW
Invalid         DROP

Everything else DROP
```

## Established / Related

```routeros
/ip firewall filter add \
chain=forward \
action=accept \
connection-state=established,related,untracked \
comment="FORWARD - established, related"
```

## Invalid

```routeros
/ip firewall filter add \
chain=forward \
action=drop \
connection-state=invalid \
comment="FORWARD - drop invalid"
```

## Block WAN → LAN

```routeros
/ip firewall filter add \
chain=forward \
action=drop \
in-interface-list=WAN \
out-interface-list=LAN \
comment="FORWARD - block WAN to LAN"
```

## Allow LAN → WAN

```routeros
/ip firewall filter add \
chain=forward \
action=accept \
in-interface-list=LAN \
out-interface-list=WAN \
comment="FORWARD - LAN to Internet"
```

## Default Forward Drop

```routeros
/ip firewall filter add \
chain=forward \
action=drop \
comment="FORWARD - drop everything else"
```

---

# ⚠️ 13. Firewall Rule Order

RouterOS processes rules from top to bottom **within each chain**.

Verify the actual order:

```routeros
/ip firewall filter print
```

The intended order is:

```text
INPUT
 ├─ established, related
 ├─ drop invalid
 ├─ allow LAN
 ├─ log new WAN connections
 ├─ drop WAN
 └─ drop everything else


FORWARD
 ├─ established, related
 ├─ drop invalid
 ├─ block WAN → LAN
 ├─ allow LAN → WAN
 └─ drop everything else
```

> ⚠️ The `LAN → WAN` accept rule must appear before the final `FORWARD` drop rule.

---

# 🌐 14. NAT

NAT is required for LAN clients using private addresses to access the Internet.

## ⭐ Starlink

```routeros
/ip firewall nat add \
chain=srcnat \
action=masquerade \
out-interface=ether1 \
comment="NAT - Starlink"
```

## 📡 Microwave

```routeros
/ip firewall nat add \
chain=srcnat \
action=masquerade \
out-interface=ether4 \
comment="NAT - Microwave Internet"
```

Verify:

```routeros
/ip firewall nat print
```

Expected:

```text
NAT - Starlink
NAT - Microwave Internet
```

---

# 🕐 15. System Time and NTP

Correct system time is important for:

* 🔥 Firewall logs
* 🔑 Authentication events
* 📝 System logs
* 📡 WAN monitoring
* 🔄 Failover monitoring
* ⏰ Scheduled tasks
* 🛠️ Troubleshooting
* 💾 Backup schedules

## Check Clock

```routeros
/system clock print
```

## Set Kyiv Time Zone

```routeros
/system clock set time-zone-name=Europe/Kyiv
```

## Enable NTP

```routeros
/system ntp client set enabled=yes
```

## Add NTP Servers

```routeros
/system ntp client servers add address=time.cloudflare.com
```

```routeros
/system ntp client servers add address=pool.ntp.org
```

Verify:

```routeros
/system ntp client print
```

```routeros
/system clock print
```

If WAN connectivity is unavailable, an NTP status such as:

```text
status: waiting
```

can be expected.

---

# 📶 16. Wi-Fi Monitoring

The hAP ax³ uses:

```text
wifi1 → 2.4 GHz
wifi2 → 5 GHz
```

## Check Interfaces

```routeros
/interface wifi print detail
```

## Connected Clients

```routeros
/interface wifi registration-table print detail
```

## Client Statistics

```routeros
/interface wifi registration-table print stats
```

## Monitor 2.4 GHz

```routeros
/interface monitor-traffic wifi1
```

Stop with:

```text
Ctrl+C
```

## Monitor 5 GHz

```routeros
/interface monitor-traffic wifi2
```

## Wi-Fi Configuration

```routeros
/interface wifi configuration print detail
```

## Wi-Fi Security

```routeros
/interface wifi security print detail
```

## Wi-Fi Datapath

```routeros
/interface wifi datapath print detail
```

Expected datapath:

```text
wifi-datapath
bridge=bridgeLocal
```

---

# 📶 17. Wi-Fi Network

Current wireless topology:

```text
                    MikroTik hAP ax³
                           │
             ┌─────────────┴─────────────┐
             │                           │
          wifi1                        wifi2
        2.4 GHz                        5 GHz
             │                           │
          SSID: Orion                 SSID: Orion
             │                           │
             └─────────────┬─────────────┘
                           │
                    wifi-datapath
                           │
                     bridgeLocal
                           │
                  192.168.88.0/24
```

Both wireless interfaces use the same LAN bridge.

> 🔐 The Wi-Fi password is intentionally not documented here.

---

# 🛡️ 18. MAC Management Security

Restrict MAC-based management to LAN:

```routeros
/tool mac-server set allowed-interface-list=LAN
```

Restrict MAC WinBox:

```routeros
/tool mac-server mac-winbox set allowed-interface-list=LAN
```

Verify:

```routeros
/tool mac-server print
```

```routeros
/tool mac-server mac-winbox print
```

Expected:

```text
allowed-interface-list: LAN
```

---

# 🔎 19. Neighbor Discovery

Restrict Neighbor Discovery to LAN:

```routeros
/ip neighbor discovery-settings set discover-interface-list=LAN
```

Verify:

```routeros
/ip neighbor discovery-settings print
```

Expected:

```text
discover-interface-list: LAN
```

This keeps Neighbor Discovery off the WAN interfaces.

---

# 👤 20. Router User

The default `admin` account was replaced with a dedicated administrator account.

Current account:

```text
User: ****
Group: full
```

Verify:

```routeros
/user print
```

> 🔐 Do not publish the administrator password in GitHub.

---

# 💾 21. Configuration Backup

Create a binary RouterOS backup:

```routeros
/system backup save name=hap-ax3-before-dualwan
```

Result:

```text
hap-ax3-before-dualwan.backup
```

This is intended for restoring the RouterOS configuration.

---

# 📄 22. Text Configuration Export

Create a human-readable export:

```routeros
/export file=hap-ax3-before-dualwan
```

Result:

```text
hap-ax3-before-dualwan.rsc
```

The `.rsc` export is useful for:

* Documentation
* Configuration review
* GitHub
* Troubleshooting
* Manual recovery
* Configuration history

### Backup vs Export

| File      | Purpose                               |
| --------- | ------------------------------------- |
| `.backup` | Binary RouterOS configuration backup  |
| `.rsc`    | Human-readable RouterOS configuration |

> ⚠️ Before publishing an `.rsc` file, inspect it for passwords, keys, certificates and other secrets.

---

# 🔍 23. Verification Commands

## Interfaces

```routeros
/interface print
```

## Bridge Ports

```routeros
/interface bridge port print
```

## IP Addresses

```routeros
/ip address print
```

## DHCP Clients

```routeros
/ip dhcp-client print detail
```

## DHCP Server

```routeros
/ip dhcp-server print
```

## DHCP Leases

```routeros
/ip dhcp-server lease print
```

## Routes

```routeros
/ip route print detail
```

## DNS

```routeros
/ip dns print
```

## NAT

```routeros
/ip firewall nat print
```

## Firewall

```routeros
/ip firewall filter print
```

## Firewall Details

```routeros
/ip firewall filter print detail
```

## Active Connections

```routeros
/ip firewall connection print
```

## Users

```routeros
/user print
```

## Services

```routeros
/ip service print
```

## Backup Files

```routeros
/file print
```

---

# ⭐ 24. Connect Starlink

Connect:

```text
Starlink
   │
   ▼
ether1
```

Leave `ether4` disconnected initially.

Wait approximately 30–60 seconds.

Check:

```routeros
/ip dhcp-client print detail
```

Look for:

```text
status=bound
address=...
gateway=...
```

Then:

```routeros
/ip address print
```

And:

```routeros
/ip route print detail
```

Test:

```routeros
/ping 1.1.1.1
```

Test DNS:

```routeros
/ping google.com
```

Record the actual values:

```text
Starlink IP:
Starlink Gateway:
Starlink DHCP Status:
Starlink Default Route:
Ping 1.1.1.1:
```

> ⚠️ Do not manually use example gateway addresses. Record the values actually provided by Starlink.

---

# 📡 25. Test Microwave Internet

Disconnect Starlink temporarily.

Connect:

```text
Microwave Internet
        │
        ▼
      ether4
```

Check:

```routeros
/ip dhcp-client print detail
```

```routeros
/ip address print
```

```routeros
/ip route print detail
```

Test:

```routeros
/ping 1.1.1.1
```

Record:

```text
Microwave IP:
Microwave Gateway:
Microwave DHCP Status:
Microwave Default Route:
Ping 1.1.1.1:
```

---

# 🔄 26. Basic Dual-WAN Failover

The initial routing priority is:

```text
Starlink
distance=1
PRIMARY
```

```text
Microwave
distance=2
BACKUP
```

Normal operation:

```text
LAN
 │
 ▼
MikroTik
 │
 ▼
Starlink
 │
 ▼
Internet
```

If Starlink fails:

```text
LAN
 │
 ▼
MikroTik
 │
 ├── X Starlink
 │
 ▼
Microwave
 │
 ▼
Internet
```

When the primary route becomes active again, RouterOS can prefer the lower-distance Starlink route.

---

# ⚠️ 27. Gateway Failure vs Internet Failure

A simple route distance does not necessarily detect every Internet outage.

For example:

```text
MikroTik → Starlink Gateway
```

may remain reachable while:

```text
Starlink → Internet
```

is unavailable.

For reliable failover, the final configuration should use an Internet reachability mechanism such as:

* Recursive routing
* `check-gateway`
* Appropriate RouterOS monitoring

The actual configuration should be created after the real WAN gateways are known.

Do not assume gateways such as:

```text
192.168.1.1
10.10.10.1
```

unless the connected equipment actually uses them.

---

# 🔀 28. Failover vs PCC Load Balancing

These are two different operating modes.

## Failover

```text
Starlink       → PRIMARY
Microwave      → BACKUP
```

One WAN carries normal traffic while the other remains available as backup.

---

## PCC 50/50 Load Balancing

With PCC:

```text
LAN connections
       │
       ├── ~50% → Starlink
       │
       └── ~50% → Microwave
```

When both WANs are available, connections can be distributed between them.

If one WAN fails, the remaining WAN can carry the traffic.

> ℹ️ PCC distributes **connections**, not the bandwidth of one individual TCP connection. A single download/TCP session normally remains associated with one WAN.

---

# 🔀 29. Planned PCC Architecture

The final dual-WAN design will use:

```text
                    LAN
                     │
                     ▼
                  MikroTik
                     │
              ┌──────┴──────┐
              │             │
             PCC           PCC
              │             │
         Starlink       Microwave
           ether1          ether4
```

The configuration will use:

* Connection marking
* Routing marks
* Separate routing paths
* WAN reachability checks
* NAT for both WANs
* Automatic failover

The exact PCC configuration should be applied only after the actual WAN addressing and gateways are known.

---

# 🧪 30. Final Testing Plan

## Test 1 — Starlink Only

```text
Starlink   → UP
Microwave  → DOWN
```

Expected:

```text
Internet → working
```

---

## Test 2 — Microwave Only

```text
Starlink   → DOWN
Microwave  → UP
```

Expected:

```text
Internet → working
```

---

## Test 3 — Both WANs

```text
Starlink   → UP
Microwave  → UP
```

Expected initial routing:

```text
Starlink → primary
Microwave → backup
```

---

## Test 4 — Starlink Failure

```text
Starlink   → DOWN
Microwave  → UP
```

Expected:

```text
Traffic → Microwave
```

---

## Test 5 — Starlink Recovery

```text
Starlink   → UP
Microwave  → UP
```

Expected routing preference:

```text
Starlink → distance 1
Microwave → distance 2
```

---

# 📋 31. Current Configuration Summary

```text
MikroTik hAP ax³
RouterOS 7.12.2

WAN1:
ether1 → Starlink
DHCP client
distance=1
NAT masquerade

WAN2:
ether4 → Microwave Internet
DHCP client
distance=2
NAT masquerade

LAN:
ether2
ether3
ether5

LAN bridge:
bridgeLocal

LAN IP:
192.168.88.1/24

DHCP pool:
192.168.88.10-192.168.88.254

DNS:
1.1.1.1
8.8.8.8

DHCP Server:
LAN-DHCP

Wi-Fi:
wifi1 → 2.4 GHz
wifi2 → 5 GHz
SSID → Orion

Management:
WinBox → LAN
SSH → LAN
FTP → LAN

MAC management:
LAN only

Neighbor Discovery:
LAN only

Firewall:
INPUT protected
FORWARD protected
WAN → LAN blocked
LAN → WAN allowed

Backups:
hap-ax3-before-dualwan.backup
hap-ax3-before-dualwan.rsc
```

---

# 🧭 32. Configuration Roadmap

```text
Device Identification
        ↓
Port Assignment
        ↓
LAN Configuration
        ↓
DHCP
        ↓
DNS
        ↓
Interface Lists
        ↓
Firewall
        ↓
NAT
        ↓
Management Hardening
        ↓
MAC Management Restriction
        ↓
Neighbor Discovery Restriction
        ↓
NTP
        ↓
Wi-Fi Configuration / Monitoring
        ↓
Backup + RSC Export
        ↓
        ★
   Connect WANs
        ↓
Collect Real WAN IPs/Gateways
        ↓
Test Starlink
        ↓
Test Microwave
        ↓
Dual-WAN Routing
        ↓
Internet Reachability Checks
        ↓
PCC 50/50
        ↓
Automatic Failover
        ↓
Final Dual-WAN Testing
```

---

# 🛡️ Security Checklist

Before publishing the configuration:

```text
☑ Remove administrator passwords
☑ Remove Wi-Fi passwords
☑ Remove private keys
☑ Remove VPN credentials
☑ Remove certificates containing private material
☑ Review exported .rsc files
☑ Keep binary backups private
☑ Restrict management to LAN
☑ Disable unused services
☑ Restrict MAC management to LAN
☑ Restrict Neighbor Discovery to LAN
☑ Log blocked WAN input
☑ Maintain a known-good backup
```

---

# 🚀 Next Step

After physically connecting the WANs, collect the real provider information.

For Starlink:

```routeros
/ip dhcp-client print detail
/ip address print
/ip route print detail
/ping 1.1.1.1
```

For the microwave link:

```routeros
/ip dhcp-client print detail
/ip address print
/ip route print detail
/ping 1.1.1.1
```

These real values are required before building the final **PCC + automatic failover** configuration.

---

# 🔗 Useful Links

* [MikroTik RouterOS Documentation](https://help.mikrotik.com/docs/?utm_source=chatgpt.com)
* [MikroTik WiFi Documentation](https://help.mikrotik.com/docs/display/ROS/WiFi?utm_source=chatgpt.com)
* [MikroTik hAP ax³](https://mikrotik.com/product/hap_ax3?utm_source=chatgpt.com)
* [Cloudflare Time Services](https://www.cloudflare.com/time/?utm_source=chatgpt.com)
* [NTP Pool Project](https://www.pool.ntp.org/?utm_source=chatgpt.com)
