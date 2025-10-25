# README — npm with Nexus (Hesaba) on Ubuntu

This is the **final, minimal-but-complete** guide to use npm with **authentication** against your Nexus.

## Repositories
- **Install (Group):** `https://packages.hesaba.co/repository/hesaba-npm/`
- **Publish (Hosted):** `https://packages.hesaba.co/repository/hesaba-npm-ito/`
- **Proxy (npmjs):** `https://packages.hesaba.co/repository/hesaba-npm-npmjs/` *(already included behind the Group; do not use directly)*

> Always use the **Group** for installs and the **Hosted** for publishing.

---

## 0) Prerequisites (on Nexus)
- A user (or user token) with **browse/read** on `hesaba-npm` (Group).
- For publishing to `hesaba-npm-ito` (Hosted), grant **add/edit** on that repo.
- If Nexus uses a private CA, have your CA file path ready (e.g., `/usr/local/share/ca-certificates/hesaba-ca.crt`).

---

## 1) Ubuntu — Create `~/.npmrc` for **installs** (Group)

```bash
# Replace with your credentials
NPM_USER='YOUR_USER'
NPM_PASS='YOUR_PASSWORD_OR_TOKEN'

# Base64 without newline (Linux). On macOS, drop -w0 if it errors.
NPM_PASS_B64="$(printf %s "$NPM_PASS" | base64 -w0 2>/dev/null || printf %s "$NPM_PASS" | base64)"

# Write ~/.npmrc  (TRAILING SLASH in URLs IS REQUIRED)
cat > ~/.npmrc <<EOF
registry=https://packages.hesaba.co/repository/hesaba-npm/
always-auth=true
//packages.hesaba.co/repository/hesaba-npm/:username=$NPM_USER
//packages.hesaba.co/repository/hesaba-npm/:_password=$NPM_PASS_B64
//packages.hesaba.co/repository/hesaba-npm/:email=you@example.com
EOF

chmod 600 ~/.npmrc
```

**Test installs:**
```bash
npm ping --registry=https://packages.hesaba.co/repository/hesaba-npm/
npm view lodash version --registry=https://packages.hesaba.co/repository/hesaba-npm/
```

> Using `sudo`? npm then reads `/root/.npmrc`. Copy your file if needed:
```bash
sudo cp ~/.npmrc /root/.npmrc && sudo chmod 600 /root/.npmrc
```

---

## 2) (Optional) Configure **publishing** to Hosted (`hesaba-npm-ito`)

Add Hosted auth to your user config (keeps credentials out of your project repo):
```bash
cat >> ~/.npmrc <<EOF

# Auth for Hosted (publish)
 //packages.hesaba.co/repository/hesaba-npm-ito/:username=$NPM_USER
 //packages.hesaba.co/repository/hesaba-npm-ito/:_password=$NPM_PASS_B64
EOF
```

In your project, point your **scope** to Hosted (safe to commit):
```ini
# ./.npmrc (next to package.json)
@your-scope:registry=https://packages.hesaba.co/repository/hesaba-npm-ito/
```

Publish:
```bash
npm publish --dry-run --registry=https://packages.hesaba.co/repository/hesaba-npm-ito/
# Real publish:
# npm publish --registry=https://packages.hesaba.co/repository/hesaba-npm-ito/
```

> If you don’t use a scope, pass `--registry` on publish as shown above.

---

## 3) Private CA (if Nexus uses internal certs)
```bash
# System-wide for npm
npm config set cafile /usr/local/share/ca-certificates/hesaba-ca.crt
# Or only for this shell/session
export NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/hesaba-ca.crt
```

---

## 4) Troubleshooting (quick)
- **E401 / Unauthorized** — wrong `username/_password` (Base64!), `always-auth=true` missing, or the user lacks *browse/read*.
- **E403 / Forbidden** — missing *add/edit* on the Hosted repo for publish.
- **self signed certificate** — set `cafile` or `NODE_EXTRA_CA_CERTS` (see §3).
- **404 / not found** — wrong URL or missing trailing slash (`.../hesaba-npm/`, `.../hesaba-npm-ito/`).
- **Using sudo** — npm may read `/root/.npmrc`; copy your file for root.

---

## TL;DR
1) Put credentials in `~/.npmrc` pointing to the **Group**:
```ini
registry=https://packages.hesaba.co/repository/hesaba-npm/
always-auth=true
//packages.hesaba.co/repository/hesaba-npm/:username=YOUR_USER
//packages.hesaba.co/repository/hesaba-npm/:_password=BASE64_OF_PASSWORD_OR_TOKEN
```
2) (Optional) For **publish**, add Hosted auth to `~/.npmrc` and set your project scope to `hesaba-npm-ito`.
3) Test with `npm ping` and `npm view` using the Group URL.
