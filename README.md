# 🌐 MikroTik CAPsMAN Setup (RouterOS **v7.20.2**) — **hEX S** + **cAP ax**

> **Topology**
```
Internet/Modem → hEX S (ether1 = WAN)
                       |
                   hEX S (ether2 = LAN)
                       |
                  cAP ax (ether1)
```

---

## 🚦 hEX S (Controller, DHCP, NAT, CAPsMAN)

> ⚠️ Optional: start clean (this wipes config)
```bash
/system reset-configuration no-defaults=yes
```

### 🧩 Bridge + LAN
```bash
/interface bridge add name=bridge-lan
/interface bridge port add bridge=bridge-lan interface=ether2
/interface bridge port add bridge=bridge-lan interface=ether3
/interface bridge port add bridge=bridge-lan interface=ether4
/interface bridge port add bridge=bridge-lan interface=ether5
```

### 🆔 LAN IP + DHCP + DNS
```bash
/ip address add address=192.168.88.1/24 interface=bridge-lan
/ip pool add name=pool-lan ranges=192.168.88.10-192.168.88.254
/ip dhcp-server add name=dhcp-lan interface=bridge-lan address-pool=pool-lan disabled=no
/ip dhcp-server network add address=192.168.88.0/24 gateway=192.168.88.1 dns-server=192.168.88.1
/ip dns set allow-remote-requests=yes
```

### 🌍 WAN (DHCP client) + NAT
```bash
/ip dhcp-client add interface=ether1 use-peer-dns=yes use-peer-ntp=yes
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

### 🛡️ Minimal firewall (safe-by-default)
```bash
# INPUT: allow established/related; allow from LAN; drop the rest
/ip firewall filter add chain=input connection-state=established,related action=accept
/ip firewall filter add chain=input connection-state=invalid action=drop
/ip firewall filter add chain=input in-interface=bridge-lan action=accept
/ip firewall filter add chain=input action=drop

# FORWARD: allow established/related; allow LAN to anywhere; drop the rest
/ip firewall filter add chain=forward connection-state=established,related action=accept
/ip firewall filter add chain=forward in-interface=bridge-lan action=accept
/ip firewall filter add chain=forward action=drop
```

### 📡 CAPsMAN (new WiFi stack in v7)
> Replace the placeholders below:
> - **SSID** → `MyHomeWiFi`
> - **Passphrase** → `YourStrongPassword`

```bash
# 1) WiFi Security
/interface wifi security add name=wifi-sec authentication-types=wpa2-psk,wpa3-psk passphrase="YourStrongPassword"

# 2) Datapath → tells CAPs to bridge Wi‑Fi into a bridge named "bridge" on each CAP
/interface wifi datapath add name=dp-cap bridge=bridge

# 3) Configs (same SSID for both bands)
/interface wifi configuration
add name=cfg-5g ssid="MyHomeWiFi" security=wifi-sec
add name=cfg-2g ssid="MyHomeWiFi" security=wifi-sec

# 4) Provisioning rules (auto-create for proper bands)
/interface wifi provisioning
add action=create-dynamic-enabled master-configuration=cfg-5g supported-bands=5ghz-ax,5ghz-ac datapath=dp-cap
add action=create-dynamic-enabled master-configuration=cfg-2g supported-bands=2ghz-ax,2ghz-n datapath=dp-cap

# 5) Enable CAPsMAN on LAN
/interface wifi capsman set enabled=yes interfaces=bridge-lan ca-certificate=auto
```

---

## 📶 cAP ax (CAP, bridged AP managed by CAPsMAN)

> ⚠️ Optional: start clean (this wipes config)
```bash
/system reset-configuration no-defaults=yes
```

### 🧩 Local bridge (wire + target datapath)
> The **bridge name must be `bridge`** to match the controller datapath.
```bash
/interface bridge add name=bridge
/interface bridge port add bridge=bridge interface=ether1
/interface bridge port add bridge=bridge interface=ether2
```

### 🔌 Get IP from hEX (for management)
```bash
/ip dhcp-client add interface=bridge disabled=no
```

### 🛣️ Datapath used by local radios
```bash
/interface wifi datapath add name=capdp bridge=bridge
```

### 🤝 Hand radios to CAPsMAN
```bash
# Make radios managed by CAPsMAN and attach local datapath
/interface wifi set [find default-name=wifi1] configuration.manager=capsman datapath=capdp disabled=no
/interface wifi set [find default-name=wifi2] configuration.manager=capsman datapath=capdp disabled=no

# Enable CAP discovery towards the hEX S
/interface wifi cap set enabled=yes discovery-interfaces=bridge caps-man-addresses=192.168.88.1
```

---

## ✅ Quick checks
```bash
# On hEX S
/interface wifi remote-cap print          # cAP should appear
/interface wifi interfaces print          # dynamic Wi‑Fi interfaces created
/ip dhcp-server lease print               # clients getting 192.168.88.x

# Sanity
/ip firewall nat print where action=masquerade
/ip route print where dst-address=0.0.0.0/0
/ip dns print   # allow-remote-requests: yes
```

---

## 🧯 Troubleshooting tips
- No IP on clients? Ensure the CAP’s radios use a datapath that bridges into a **bridge named `bridge`** on the CAP, and that `ether1` is a port of that bridge.
- “Connected, no internet”? Check DNS on hEX (`/ip dns set allow-remote-requests=yes`) or set public DNS in `/ip dhcp-server network`.
- IP range conflict with modem? Change LAN to another subnet (e.g., `192.168.50.0/24`).

---

## ➕ Adding more APs later
1) Reset the new AP (`no-defaults`).  
2) Create `bridge`, add `ether1/ether2` as ports.  
3) Add `dhcp-client` on `bridge`.  
4) Set radios to `configuration.manager=capsman`, add a local datapath to `bridge`, enable `/interface wifi cap`.  
It will **auto-provision** from CAPsMAN and start broadcasting your SSID. ✨
