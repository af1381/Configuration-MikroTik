# README — npm with Nexus (Hesaba)

This guide shows **exact, minimal steps** to use npm with authentication against your Nexus repositories:

- **Group (install):** `https://packages.hesaba.co/repository/hesaba-npm/`
- **Hosted (publish):** `https://packages.hesaba.co/repository/hesaba-npm-ito/`
- **Proxy to npmjs (info only):** `https://packages.hesaba.co/repository/hesaba-npm-npmjs/` (already included behind the Group)
- (Repeat) Group: `https://packages.hesaba.co/repository/hesaba-npm/`

> Use the **Group** for installs. Use the **Hosted** for publishing. You normally do **not** point to the proxy directly.

---

## 0) Prerequisites (Nexus)
- A Nexus user (or user token) with **browse/read** on `hesaba-npm` (Group).  
- For publishing to `hesaba-npm-ito` (Hosted), also grant **add/edit** on that repo.
- Optional: If Nexus uses a private CA, have your CA file path ready.

---

## 1) Ubuntu — Create `~/.npmrc` for **installing** packages (Group)
> Credentials are Base64-encoded (npm Basic Auth). The trailing slash on URLs **matters**.

```bash
# Replace these with your real values
NPM_USER='YOUR_USER'
NPM_PASS='YOUR_PASSWORD_OR_TOKEN'

# Base64 without newline (Linux). On macOS, drop -w0 if it errors.
NPM_PASS_B64="$(printf %s "$NPM_PASS" | base64 -w0 2>/dev/null || printf %s "$NPM_PASS" | base64)"

# Create ~/.npmrc
cat > ~/.npmrc <<EOF
registry=https://packages.hesaba.co/repository/hesaba-npm/
always-auth=true
//packages.hesaba.co/repository/hesaba-npm/:username=$NPM_USER
//packages.hesaba.co/repository/hesaba-npm/:_password=$NPM_PASS_B64
//packages.hesaba.co/repository/hesaba-npm/:email=you@example.com
EOF

chmod 600 ~/.npmrc
```

**Test:**
```bash
npm ping --registry=https://packages.hesaba.co/repository/hesaba-npm/
npm view lodash version --registry=https://packages.hesaba.co/repository/hesaba-npm/
```

> If you run npm with `sudo`, it reads `/root/.npmrc`. Copy your file for root if needed:  
> `sudo cp ~/.npmrc /root/.npmrc && sudo chmod 600 /root/.npmrc`

---

## 2) (Optional) Configure **publishing** to Hosted (`hesaba-npm-ito`)
**A.** Add Hosted auth to your user config (keeps credentials out of your project repo):
```bash
cat >> ~/.npmrc <<EOF

# Auth for Hosted (publish)
# (Credentials reused from above)
 //packages.hesaba.co/repository/hesaba-npm-ito/:username=$NPM_USER
 //packages.hesaba.co/repository/hesaba-npm-ito/:_password=$NPM_PASS_B64
EOF
```

**B.** In your project, point your **scope** to the Hosted registry (safe to commit):
```ini
# ./.npmrc  (placed next to package.json)
@your-scope:registry=https://packages.hesaba.co/repository/hesaba-npm-ito/
```

**Publish:**
```bash
npm publish --dry-run --registry=https://packages.hesaba.co/repository/hesaba-npm-ito/
# Real publish:
# npm publish --registry=https://packages.hesaba.co/repository/hesaba-npm-ito/
```

> If you don’t use a scope, pass `--registry` on the publish command as shown.

---

## 3) CI/CD variant (no files needed)
```bash
# Assume NPM_USER and NPM_PASSWORD are CI secrets
NPM_PASS_B64="$(printf %s "$NPM_PASSWORD" | base64 -w0 2>/dev/null || printf %s "$NPM_PASSWORD" | base64)"
npm config set registry https://packages.hesaba.co/repository/hesaba-npm/
npm config set always-auth true
npm config set //packages.hesaba.co/repository/hesaba-npm/:username "$NPM_USER"
npm config set //packages.hesaba.co/repository/hesaba-npm/:_password "$NPM_PASS_B64"

# For publishing (optional)
npm config set @your-scope:registry https://packages.hesaba.co/repository/hesaba-npm-ito/
npm config set //packages.hesaba.co/repository/hesaba-npm-ito/:username "$NPM_USER"
npm config set //packages.hesaba.co/repository/hesaba-npm-ito/:_password "$NPM_PASS_B64"
```

---

## 4) Private CA (if Nexus uses an internal cert)
```bash
# System-wide for npm
npm config set cafile /usr/local/share/ca-certificates/hesaba-ca.crt
# Or for the current shell only
export NODE_EXTRA_CA_CERTS=/usr/local/share/ca-certificates/hesaba-ca.crt
```

---

## 5) Troubleshooting
- **E401 / Unauthorized**: wrong `username/_password` (Base64), `always-auth=true` missing, or the user lacks *browse/read*.
- **E403 / Forbidden**: missing *add/edit* on the Hosted repo for publish.
- **self signed certificate**: set `cafile` or `NODE_EXTRA_CA_CERTS` (see §4).
- **404 / not found**: wrong URL or missing trailing slash (`.../hesaba-npm/`, `.../hesaba-npm-ito/`).
- **Using sudo**: npm may read `/root/.npmrc`; copy your file for root.

---

### Notes on the third URL (Proxy)
`hesaba-npm-npmjs` is a **proxy** of npmjs.org, already included inside the **Group** (`hesaba-npm`). You typically **do not** need to point clients at it directly.
