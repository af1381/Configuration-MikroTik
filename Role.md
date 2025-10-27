# =====================================
# 🔒 MikroTik Firewall Rules for hEX + CAPsMAN
# 🧱 Scenario: 1×WAN (ether1) + 1×LAN (bridge-lan)
# =====================================

# -------------------------------
# ⚙️  Variables (Edit if needed)
# -------------------------------
:local WAN "ether1"
:local LAN "bridge-lan"

# -------------------------------
# 🧠 INPUT CHAIN  (Traffic TO router)
# -------------------------------

# 1️⃣ Allow established & related
/ip firewall filter add chain=input connection-state=established,related action=accept comment="✅ Allow established/related"

# 2️⃣ Drop invalid packets
/ip firewall filter add chain=input connection-state=invalid action=drop comment="🚫 Drop invalid connections"

# 3️⃣ Allow management & ping from LAN
/ip firewall filter add chain=input in-interface=$LAN protocol=tcp dst-port=22,8291,80,443 action=accept comment="✅ Allow management from LAN (SSH/WinBox/Web)"
/ip firewall filter add chain=input in-interface=$LAN protocol=icmp action=accept comment="✅ Allow ping from LAN"

# 4️⃣ Allow CAPsMAN discovery & CAPs (needed for CAPsMAN)
/ip firewall filter add chain=input in-interface=$LAN protocol=udp dst-port=5246,5247 action=accept comment="✅ Allow CAPsMAN control/forward ports"

# 5️⃣ Allow DHCP requests
/ip firewall filter add chain=input protocol=udp dst-port=67,68 action=accept comment="✅ Allow DHCP client/server"

# 6️⃣ Allow DNS queries (since DNS allow-remote-requests=yes)
/ip firewall filter add chain=input protocol=udp dst-port=53 action=accept comment="✅ Allow DNS queries"

# 7️⃣ Drop all other inbound from WAN
/ip firewall filter add chain=input in-interface=$WAN action=drop comment="🚫 Drop all inbound from WAN"

# -------------------------------
# 🌉 FORWARD CHAIN  (Traffic THROUGH router)
# -------------------------------

# 1️⃣ Allow established & related
/ip firewall filter add chain=forward connection-state=established,related action=accept comment="✅ Allow established/related forwarding"

# 2️⃣ Drop invalid
/ip firewall filter add chain=forward connection-state=invalid action=drop comment="🚫 Drop invalid forwarding"

# 3️⃣ Allow LAN → WAN
/ip firewall filter add chain=forward in-interface=$LAN out-interface=$WAN connection-state=new action=accept comment="✅ Allow LAN to WAN"

# 4️⃣ Allow CAPs traffic (LAN↔LAN)
/ip firewall filter add chain=forward in-interface=$LAN out-interface=$LAN action=accept comment="✅ Allow LAN↔LAN (CAPs ↔ Clients)"

# 5️⃣ Drop all other forwarding (default deny)
/ip firewall filter add chain=forward action=drop comment="🚫 Drop all other forwarding"

# -------------------------------
# 🔁 NAT Rules
# -------------------------------

# 1️⃣ Masquerade for Internet
/ip firewall nat add chain=srcnat out-interface=$WAN action=masquerade comment="🌐 Masquerade LAN → WAN"

# -------------------------------
# 🧩 Optional Protections
# -------------------------------

# Drop brute-force attempts (SSH/Winbox)
/ip firewall filter add chain=input in-interface=$WAN protocol=tcp dst-port=22,8291 connection-limit=3,32 action=drop comment="🚫 Limit SSH/WinBox brute force"

# Limit ICMP (prevent ping flood)
/ip firewall filter add chain=input protocol=icmp limit=50/5s,2 action=accept comment="✅ Allow limited ping"
/ip firewall filter add chain=input protocol=icmp action=drop comment="🚫 Drop excessive ping"

# Drop invalid fragments
/ip firewall filter add chain=input fragment=yes action=drop comment="🚫 Drop fragmented packets"

# -------------------------------
# ✅ Done
# -------------------------------
:log info "🔥 Firewall rules for hEX + CAPsMAN applied successfully!"