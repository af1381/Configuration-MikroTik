# README — Configure APT via Nexus (`packages.hesaba.co`)

This document explains how to configure Ubuntu clients to use **Nexus** as the source for APT. Target distro in examples: **Ubuntu 22.04 (Jammy)**. For **Ubuntu 24.04 (Noble)**, see the appendix.

---

## 1) Server-side prerequisites (Nexus)

1. Go to **Administration → Security → Anonymous Access** and **disable** “Allow anonymous users to access the server.”
2. Create a read-only role in **Security → Roles** (e.g., `apt-readers`) and grant these privileges for each APT repo you want exposed:
   - `nx-repository-view-apt-apt-ubuntu-jammy-browse`
   - `nx-repository-view-apt-apt-ubuntu-jammy-read`
   - `nx-repository-view-apt-apt-ubuntu-jammy-security-browse`
   - `nx-repository-view-apt-apt-ubuntu-jammy-security-read`
   > Pattern: `nx-repository-view-<format>-<repository>-browse|read`
3. In **Security → Users**, create a client user (e.g., `aptclient`) and assign the role above.  
   **Tip:** Prefer **User Tokens** (Security → User Tokens) so you can provide a token instead of a real password.

---

## 2) Ubuntu client configuration (22.04 Jammy)

### 2.1) APT authentication (secure and clean)
Use an `auth.conf.d` entry so user/password is not embedded in repository URLs.
```bash
sudo install -m 0700 -d /etc/apt/auth.conf.d
sudo tee /etc/apt/auth.conf.d/hesaba.conf >/dev/null <<'EOF'
machine packages.hesaba.co
login aptclient
password "YOUR_TOKEN_OR_PASSWORD"
EOF
sudo chown root:root /etc/apt/auth.conf.d/hesaba.conf
sudo chmod 600 /etc/apt/auth.conf.d/hesaba.conf
```
> If you use a non-standard port, specify it in the machine name: `packages.hesaba.co:PORT`  
> Do **not** include a scheme (no `https://`) in `machine`.

### 2.2) APT sources for Jammy
`apt-ubuntu-jammy` serves `jammy`, `jammy-updates`, and `jammy-backports`.  
`apt-ubuntu-jammy-security` serves `jammy-security`.

```bash
# Optional: backup and temporarily comment stock Ubuntu sources for a clean test
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i 's/^\s*deb /# &/g' /etc/apt/sources.list

sudo tee /etc/apt/sources.list.d/hesaba-jammy.list >/dev/null <<'EOF'
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-updates main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-backports main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy-security jammy-security main restricted universe multiverse
EOF
```
**Why `signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg`?** Nexus is proxying official Ubuntu archives; packages are signed with Ubuntu’s official key—no new key is needed.

### 2.3) Update and verify
```bash
sudo apt clean
sudo apt update

# smoke test
sudo apt install -y jq

# confirm origin points at your Nexus
apt-cache policy jq | sed -n '1,15p'
```

### 2.4) HTTPS with private/self-signed CA
Install your internal CA to prevent TLS errors:
```bash
sudo cp YOUR_CA_CERT.crt /usr/local/share/ca-certificates/hesaba-ca.crt
sudo update-ca-certificates
```

---

## 3) Quick troubleshooting

- **401 Unauthorized**  
  - `/etc/apt/auth.conf.d/hesaba.conf` missing/ignored (wrong file perms—must be `600`).  
  - `machine` must match exactly (`packages.hesaba.co` or `packages.hesaba.co:PORT`).
- **403 Forbidden**  
  - User exists but lacks `browse/read` on the target repos.
- **404 Not Found**  
  - Wrong path/distribution; expect `.../repository/apt-ubuntu-jammy/dists/jammy/...`.
- **TLS/Certificate**  
  - Install your internal CA (see 2.4).
- **NO_PUBKEY / GPG**  
  - Ensure the `signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg` option is present for each `deb` line.

**Direct curl test:**
```bash
curl -I -u aptclient:YOUR_TOKEN_OR_PASSWORD \
  https://packages.hesaba.co/repository/apt-ubuntu-jammy/dists/jammy/InRelease
```

---

## 4) (Optional) Docker APT for Jammy
If you have `apt-docker-jammy` proxied in Nexus and want Docker CE packages:
```bash
# official Docker GPG key (download.docker.com is proxied by Nexus)
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

sudo tee /etc/apt/sources.list.d/hesaba-docker-jammy.list >/dev/null <<'EOF'
deb [arch=amd64 signed-by=/etc/apt/keyrings/docker.gpg] https://packages.hesaba.co/repository/apt-docker-jammy jammy stable
EOF

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

---

## Appendix A) Ubuntu 24.04 (Noble Numbat)
For 24.04 clients, replace *jammy* with *noble* and use the `apt-ubuntu-noble` and `apt-ubuntu-noble-security` repos:
```bash
sudo tee /etc/apt/sources.list.d/hesaba-noble.list >/dev/null <<'EOF'
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble-updates main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble-backports main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble-security noble-security main restricted universe multiverse
EOF
sudo apt update
```
> **Warning:** Do **not** mix `noble` repos with Ubuntu 22.04 systems.

---

## Appendix B) Alternative (less secure): credentials in URL
Not recommended (credentials may leak via logs or process lists), but possible:
```bash
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] \
https://USERNAME:PASSWORD@packages.hesaba.co/repository/apt-ubuntu-jammy jammy main
```

---

## TL;DR
1) Disable Anonymous in Nexus; create read-only role and a client user (or user token).  
2) On the client, create `/etc/apt/auth.conf.d/hesaba.conf` with `machine packages.hesaba.co`.  
3) Point `jammy` and `jammy-security` sources to Nexus.  
4) `apt update` and install a test package; use the troubleshooting section if needed.
