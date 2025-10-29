# 🌐 CAPsMAN Guide — RouterOS **v7.20.2** | **hEX S** (Controller) + **two cAP ax** (CAP‑1, CAP‑2)

This README matches **exactly your setup**: **1× hEX S** and **2× cAP ax**. It also includes configuring an **ISP static /30** on the WAN (ether1) so you have real Internet.  
Best practice: CAPs do **not** need public IPs; give the public IP to hEX S and NAT the LAN/Wi‑Fi clients.

---

## 🖼️ Topology
```
Internet/ISP (/30 static) → hEX S (ether1 = WAN, Public IP)
                                 |
                             hEX S (ether2..5 = LAN, bridge-lan, 192.168.88.0/24)
                                 |
                   +-------------+-------------+
                   |                           |
               cAP ax (CAP-1)             cAP ax (CAP-2)
```

---

## 0) Prereqs
- Controller datapath uses `bridge=bridge`, so **each CAP must have a local bridge named `bridge`**.
- Firewall is “safe-by-default”: management only from LAN; other input is dropped.

---

## 1) hEX S (controller) — if not already configured
```rsc
# Bridge + LAN
/interface bridge add name=bridge-lan
/interface bridge port add bridge=bridge-lan interface=ether2
/interface bridge port add bridge=bridge-lan interface=ether3
/interface bridge port add bridge=bridge-lan interface=ether4
/interface bridge port add bridge=bridge-lan interface=ether5

# LAN IP + DHCP + DNS
/ip address add address=192.168.88.1/24 interface=bridge-lan
/ip pool add name=pool-lan ranges=192.168.88.10-192.168.88.254
/ip dhcp-server add name=dhcp-lan interface=bridge-lan address-pool=pool-lan disabled=no
/ip dhcp-server network add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=192.168.88.1
/ip dns set allow-remote-requests=yes

# WAN (DHCP client) + NAT  ← if you have a /30, apply the static section below instead
/ip dhcp-client add interface=ether1 use-peer-dns=yes use-peer-ntp=yes
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade

# Minimal firewall
/ip firewall filter add chain=input connection-state=established,related action=accept
/ip firewall filter add chain=input connection-state=invalid action=drop
/ip firewall filter add chain=input in-interface=bridge-lan action=accept
/ip firewall filter add chain=input action=drop
/ip firewall filter add chain=forward connection-state=established,related action=accept
/ip firewall filter add chain=forward in-interface=bridge-lan action=accept
/ip firewall filter add chain=forward action=drop

# CAPsMAN (wifiwave2) — change SSID/passphrase
/interface wifi security add name=wifi-sec authentication-types=wpa2-psk,wpa3-psk passphrase="YourStrongPassword"
/interface wifi datapath add name=dp-cap bridge=bridge           # reminder: each CAP must have a local bridge named "bridge"
/interface wifi configuration add name=cfg-5g ssid="MyHomeWiFi" security=wifi-sec
/interface wifi configuration add name=cfg-2g ssid="MyHomeWiFi" security=wifi-sec
/interface wifi provisioning
add action=create-dynamic-enabled master-configuration=cfg-5g supported-bands=5ghz-ax,5ghz-ac datapath=dp-cap
add action=create-dynamic-enabled master-configuration=cfg-2g supported-bands=2ghz-ax,2ghz-n datapath=dp-cap
/interface wifi capsman set enabled=yes interfaces=bridge-lan ca-certificate=auto
```

---

## 2) CAP‑1 (run on the CAP itself)
```rsc
# (Optional) factory reset
#:delay 2
#:if ([:len [/system default-configuration print]]>0) do={ /system reset-configuration no-defaults=yes skip-backup=yes }

# Identity + local bridge (name must be exactly "bridge")
/system identity set name=CAP-1
/interface bridge add name=bridge
/interface bridge port add bridge=bridge interface=ether1
/interface bridge port add bridge=bridge interface=ether2

# Get management IP from hEX S
/ ip dhcp-client add interface=bridge disabled=no

# Hand radios to CAPsMAN and enable CAP
/interface wifi set [find default-name=wifi1] configuration.manager=capsman disabled=no
/interface wifi set [find default-name=wifi2] configuration.manager=capsman disabled=no
/interface wifi cap set enabled=yes discovery-interfaces=bridge caps-man-addresses=192.168.88.1
```

---

## 3) CAP‑2 (run on the CAP itself)
```rsc
# (Optional) factory reset
#:delay 2
#:if ([:len [/system default-configuration print]]>0) do={ /system reset-configuration no-defaults=yes skip-backup=yes }

# Identity + local bridge
/system identity set name=CAP-2
/interface bridge add name=bridge
/interface bridge port add bridge=bridge interface=ether1
/interface bridge port add bridge=bridge interface=ether2

# Get management IP from hEX S
/ip dhcp-client add interface=bridge disabled=no

# Hand radios to CAPsMAN and enable CAP
/interface wifi set [find default-name=wifi1] configuration.manager=capsman disabled=no
/interface wifi set [find default-name=wifi2] configuration.manager=capsman disabled=no
/interface wifi cap set enabled=yes discovery-interfaces=bridge caps-man-addresses=192.168.88.1
```

---

## 4) WAN with ISP **static /30** — on hEX S
> Replace with real ISP values. Example:  
> **WAN IP:** `203.0.113.10/30` — **Gateway:** `203.0.113.9`

```rsc
# Disable/remove DHCP client on WAN if present
/ip dhcp-client disable [find interface=ether1]
# or:
# /ip dhcp-client remove [find interface=ether1]

# Static IP on ether1
/ip address add address=203.0.113.10/30 interface=ether1 comment="ISP /30 WAN"

# Default route to ISP gateway
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.9 comment="Default via ISP"

# (Optional) DNS
/ip dns set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes
```

### NAT (unchanged)
```rsc
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

### Connectivity tests
```rsc
/ping 203.0.113.9 src-address=203.0.113.10 count=5
/ping 1.1.1.1 count=5
/ping google.com count=5
```

---

## 5) Final checklist
- [ ] CAP‑1 and CAP‑2 are visible under **Remote CAP** and broadcasting SSIDs.  
- [ ] Default route to ISP gateway present; external ping OK.  
- [ ] Wi‑Fi clients get IPs from **192.168.88.0/24** and have Internet.  
- [ ] Management allowed only from LAN; WAN management blocked.

---

## 6) Quick troubleshooting
- **CAP not adopted?** Local bridge name must be `bridge`; cable on hEX **LAN**; controller has `input in-interface=bridge-lan accept`.  
- **Connected, no Internet?** Fix DNS on hEX or in DHCP network.  
- **Channel overlap?** Assign different 5 GHz channels to CAPs (40 MHz is a good start).
