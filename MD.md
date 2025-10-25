# README — Configure APT via Nexus (`packages.hesaba.co`) for Ubuntu 22.04 & 24.04

This file provides **client-side** steps to fetch APT packages from your Nexus at `packages.hesaba.co`.
It includes instructions for **Ubuntu 22.04 (Jammy)** and **Ubuntu 24.04 (Noble)** using the exact four `deb` lines you requested (no `signed-by`).

> Prerequisite on Nexus: you have a read-only username (or user token), with **browse/read** on the relevant APT repos. Anonymous access should be disabled.

---

## 1) APT authentication (shared for 22 & 24)
Store credentials in `auth.conf.d` so credentials are not embedded in URLs.
```bash
sudo install -m 0700 -d /etc/apt/auth.conf.d
sudo tee /etc/apt/auth.conf.d/hesaba.conf >/dev/null <<'EOF'
machine packages.hesaba.co
login aptclient
password "YOUR_TOKEN_OR_PASSWORD"
EOF
sudo chmod 600 /etc/apt/auth.conf.d/hesaba.conf
```


---

## 2) Ubuntu **22.04 (Jammy)** — add sources
> **Important:** each line must begin with `deb`. A bare URL will fail with `Type 'https://...' is not known`.

```bash
sudo tee /etc/apt/sources.list.d/hesaba.list >/dev/null <<'EOF'
deb https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy main restricted universe multiverse
deb https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-updates main restricted universe multiverse
deb https://packages.hesaba.co/repository/apt-ubuntu-jammy jammy-backports main restricted universe multiverse
deb https://packages.hesaba.co/repository/apt-ubuntu-jammy-security jammy-security main restricted universe multiverse
EOF
```

---

## 3) Ubuntu **24.04 (Noble)** — add sources
Use **Noble** only on Ubuntu 24.04. **Do not mix** Jammy and Noble on the same host.
```bash
sudo tee /etc/apt/sources.list.d/hesaba-noble.list >/dev/null <<'EOF'
deb https://packages.hesaba.co/repository/apt-ubuntu-noble noble main restricted universe multiverse
deb https://packages.hesaba.co/repository/apt-ubuntu-noble noble-updates main restricted universe multiverse
deb https://packages.hesaba.co/repository/apt-ubuntu-noble noble-backports main restricted universe multiverse
deb https://packages.hesaba.co/repository/apt-ubuntu-noble-security noble-security main restricted universe multiverse
EOF
```

---

## 4) Update & verify (both versions)
```bash
sudo apt clean
sudo rm -rf /var/lib/apt/lists/*
sudo apt update
sudo apt install -y jq

# confirm the candidate origin
apt-cache policy jq | sed -n '1,20p'
# You should see "packages.hesaba.co" in Origin/Site.
```

---

## Troubleshooting
- **Type 'https://…' is not known** — Lines in the sources file must start with `deb`, not a bare URL.
- **401 Unauthorized** — Missing/ignored `/etc/apt/auth.conf.d/hesaba.conf` or wrong file permissions (must be `600`), or hostname mismatch in `machine`.
- **403 Forbidden** — The Nexus user lacks `browse/read` on the target repos.
- **404 Not Found** — Wrong path or distribution (22.04 uses `jammy`, 24.04 uses `noble`); expect `.../dists/<codename>/...` behind the scenes.
- **NO_PUBKEY** — Switch to the secure form by adding ` [signed-by=/usr/share/keyrings/ubuntu-archive-keyring.gpg] ` after `deb`, or for quick testing use ` [trusted=yes] ` (not recommended for production).
- **TLS errors** — Install your internal CA as shown in step 1.

Useful debug:
```bash
sudo apt -o Debug::Acquire::https=true -o Debug::pkgProblemResolver=true update
```
