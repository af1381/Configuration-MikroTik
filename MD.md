# README — Configure APT via Nexus for **Ubuntu 22.04 (Jammy)** and **Ubuntu 24.04 (Noble)**

This guide shows **client-side only** steps to pull APT packages from your Nexus at `packages.hesaba.co` for **Ubuntu 22.04** and **Ubuntu 24.04**.

> Prerequisites on Nexus: you have a username/password (or user token) with *read/browse* to the APT repos, and anonymous access is disabled.

---

## 1) Configure APT authentication (both Ubuntu 22 & 24)
Store credentials in `auth.conf.d` so they are not embedded in URLs.
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

```

---

## 2) Ubuntu **22.04 (Jammy)** — set APT sources
```bash
# (Optional) clean test: backup and comment out stock sources
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i 's/^\s*deb /# &/g' /etc/apt/sources.list

sudo tee /etc/apt/sources.list.d/hesaba-jammy.list >/dev/null <<'EOF'
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-updates main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-backports main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-jammy-security jammy-security main restricted universe multiverse
EOF
```

---

## 3) Ubuntu **24.04 (Noble)** — set APT sources
```bash
# (Optional) clean test: backup and comment out stock sources
sudo cp /etc/apt/sources.list /etc/apt/sources.list.bak
sudo sed -i 's/^\s*deb /# &/g' /etc/apt/sources.list

sudo tee /etc/apt/sources.list.d/hesaba-noble.list >/dev/null <<'EOF'
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble-updates main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble noble-backports main restricted universe multiverse
deb [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] https://packages.hesaba.co/repository/apt-ubuntu-noble-security noble-security main restricted universe multiverse
EOF
```

> **Do not mix** Jammy and Noble sources on the same host. Use Jammy for 22.04 and Noble for 24.04.

---

## 4) Update & verify (both versions)
```bash
sudo apt clean
sudo apt update
sudo apt install -y jq

# verify the origin points to your Nexus
apt-cache policy jq | sed -n '1,15p'
```
You should see `packages.hesaba.co` in the output.

---

## 5) Quick troubleshooting
- **401 Unauthorized**: bad/missing `/etc/apt/auth.conf.d/hesaba.conf` or wrong file perms (must be `600`), or hostname mismatch.
- **403 Forbidden**: user lacks `browse/read` on the target repos in Nexus.
- **404 Not Found**: wrong path or distribution (`dists/jammy/...` vs `dists/noble/...`).
- **NO_PUBKEY**: ensure `signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg` is present on each `deb` line.
- **TLS errors**: install your internal CA (see step 1).

That’s all—choose section 2 **or** 3 based on your Ubuntu version, then run step 4.
