# VCF 9.1 Offline Depot — How‑To

How to build a **local offline software depot** for VMware Cloud Foundation 9.1 using the
**VCF Download Tool (VCFDT)**, host it on a web server, and point VCF at it.

This guide includes every command required. Replace the `<PLACEHOLDERS>` and `example.com` names
with values from your environment.

---

## 1. What is an offline depot, and why?

VCF normally reaches Broadcom over the internet (`dl.broadcom.com`) to download the software bundles
it needs to **install, upgrade, and patch** itself. In an **air‑gapped or restricted** environment
that path isn't available to the VCF appliances.

An **offline depot** solves this: you download all the bundles **once** (on a machine that *does*
have internet), copy them to a local web server, and tell VCF to fetch from that local server.

```
                    ┌─────────────────────┐         ┌──────────────────────────┐
  Internet  ───────▶│  VCF Download Tool  │ ──────▶ │  depot/  (local folder)  │
 (Broadcom)         │  (download host)    │ download│  bundles + metadata      │
                    └─────────────────────┘         └────────────┬─────────────┘
                                                                 │ served over HTTP/HTTPS
                                                                 ▼
                                              ┌───────────────────────────────────┐
                                              │  VCF Installer / Fleet / SDDC Mgr  │
                                              │  configured to use the offline URL │
                                              └───────────────────────────────────┘
```

**Three phases, every time:**

| Phase | What you do | Tool |
|------:|-------------|------|
| 1 | **Download** bundles + metadata | `vcf-download-tool` (VCFDT) |
| 2 | **Serve** the download folder over HTTP/HTTPS | web server / Python |
| 3 | **Point VCF** at the depot URL | VCF API (Installer / Fleet / SDDC Manager) |

---

## 2. Prerequisites

- A Linux host **with internet access** to run the downloads (the "download host").
- A **Broadcom Support Portal** account entitled to VCF (to generate the download activation code).
- **Disk space**: plan for **~150 GB+** of content for a full VCF release (install + upgrade + ESX);
  a **~500 GB** dedicated disk or mounted volume gives headroom for multiple releases/patches.
- Network reachability: the VCF appliances must be able to reach the depot **host:port**.

---

## 3. Phase 1 — Install the VCF Download Tool

1. Broadcom Support Portal: **My Downloads → VMware Cloud Foundation → (your version) →
   Drivers & Tools**, download `vcf-download-tool-<version.build>.tar.gz`.
2. Copy it to the download host and extract:
   ```bash
   tar -xzf vcf-download-tool-*.tar.gz -C /opt/
   cd /opt/vcfdt/bin
   chmod +x vcf-download-tool
   ```
3. Verify it runs:
   ```bash
   ./vcf-download-tool --version
   ./vcf-download-tool --help
   ```

Commands available: `binaries`, `metadata`, `releases`, `esx`, `depot`, `configuration`.
The tool log lives at `<install-dir>/log/vdt.log`.

---

## 4. Phase 2 — Get your Broadcom credential (token *or* activation code)

The tool authenticates with **one of two** credentials saved in a one‑line text file. Pick whichever
your account uses — **but note ESX downloads specifically require the *activation code*.**

**Option 1 — Download token** (classic, simplest). Generate a download token in the Broadcom Support
Portal and save it; used with the `--depot-download-token-file` flag:
```bash
echo '<YOUR_DOWNLOAD_TOKEN>' > /path/to/token.txt
```

**Option 2 — Activation code** (registered to this depot; **required for ESX**). Used with the
`--depot-download-activation-code-file` flag:
```bash
# 1) generate this depot's unique ID
./vcf-download-tool configuration generate --software-depot-id
# 2) at https://vcf.broadcom.com → Software Depot Registrations → New Registration,
#    paste the software-depot-id, copy the depotDownloadActivationCode, then:
echo '<YOUR_ACTIVATION_CODE>' > /path/to/activation-code.txt
```

> **Note:** Treat the token / activation‑code file as a secret — never commit it to source control.

---

## 5. Phase 3 — Download the bundles

`--sku` is `VCF` (or `VVF` for vSphere Foundation). `--vcf-version` is the **base release**, e.g.
`9.1.0`. `--depot-store` chooses where everything lands; re‑running into the same store is
**cumulative** (adds, never overwrites), and multiple releases/patches can share one store.

First, list what's available:
```bash
./vcf-download-tool binaries list --depot-download-token-file=/path/to/token.txt \
  --vcf-version=9.1.0 --sku=VCF --type=INSTALL --automated-install
```

Download the **install** bundles (fresh deploy):
```bash
./vcf-download-tool binaries download --depot-download-token-file=/path/to/token.txt \
  --depot-store=/path/to/depot --vcf-version=9.1.0 --sku=VCF --type=INSTALL --automated-install
```

Download the **upgrade** bundles (add `--upgrades-only` for *patches only*):
```bash
./vcf-download-tool binaries download --depot-download-token-file=/path/to/token.txt \
  --depot-store=/path/to/depot --vcf-version=9.1.0 --sku=VCF --type=UPGRADE
```

Download the **ESX** bundles — note this one uses the **activation code**, not the token:
```bash
./vcf-download-tool esx download \
  --depot-download-activation-code-file=/path/to/activation-code.txt \
  --depot-store=/path/to/depot
```

**Behind a proxy?** Add `--proxy-server=<FQDN:Port>` (plus `--proxy-user` /
`--proxy-user-password-file` if it needs auth; `--proxy-https` requires the proxy cert in the JRE
trust store).

> **Important:** **Use the tool — don't hand‑copy.** When done, `/path/to/depot` holds the full Broadcom depot
> layout (a `PROD/` tree **plus metadata**). VCF reads that metadata; simply copying individual
> bundle files into a folder **will not work** — you must serve the tool‑generated structure intact.

---

## 6. Phase 4 — Serve the depot over HTTP/HTTPS

VCF just needs the `--depot-store` folder **published over the network**.

> **New in VCF 9.1:** the VCF Installer and Fleet Depot Service can use a **plain HTTP** offline
> depot **with no basic auth** — but **only when configured via the API** (the 9.1 Installer **UI**
> does *not* support an HTTP offline depot). See Phase 5. ([William Lam](https://williamlam.com/2026/05/vcf-9-1-new-http-offline-depot-support-for-vcf-installer-fleet-depot-service.html))

### Option A — Simplest (HTTP, no auth)
```bash
cd /path/to/depot
python3 -m http.server 8888 --bind 0.0.0.0
# depot URL becomes:  http://depot.example.com:8888
```

### Option B — HTTP/HTTPS **with** auth (William Lam's helper)
Download `http_server_auth.py` from William Lam's scripts repo, then:
```bash
# HTTP + basic auth:
python3 http_server_auth.py --bind 0.0.0.0 --user <USER> --password '<PASSWORD>' \
  --port 8888 --directory /path/to/depot

# HTTPS + basic auth (generate a self-signed cert/key first, per his post):
python3 http_server_auth.py --bind 0.0.0.0 --user <USER> --password '<PASSWORD>' \
  --port 443 --directory /path/to/depot --certfile ~/cert.crt --keyfile ~/key.pem
```
**Reference —** Script + cert‑generation snippet: [Quick Tip — Host VCF Offline Depot with Python SimpleHTTPServer + Auth](https://williamlam.com/2025/01/quick-tip-easily-host-vmware-cloud-foundation-vcf-offline-depot-using-python-simplehttpserver-with-authentication.html)

### Keep it running with systemd
Create `/etc/systemd/system/depot-server.service`:
```ini
[Unit]
Description=VCF 9.1 offline depot
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/path/to/depot
ExecStart=/usr/bin/python3 -m http.server 8888 --bind 0.0.0.0 --directory /path/to/depot
Restart=always
RestartSec=3

[Install]
WantedBy=multi-user.target
```
Enable, start, and sanity‑check:
```bash
sudo systemctl daemon-reload
sudo systemctl enable --now depot-server.service
sudo systemctl status depot-server.service        # should be "active (running)"
curl -I http://depot.example.com:8888/
```

---

## 7. Phase 5 — Point VCF at the depot

In VCF 9.1 there are **two components that can own depot configuration, each with its own API endpoint *and* its own token** — don't reuse one component's token against the other. **HTTP offline depots must be set via the API** (the Installer UI rejects HTTP). The calls below are taken from William Lam's 9.1 walkthroughs: [VCF 9.1 — new HTTP offline depot support](https://williamlam.com/2026/05/vcf-9-1-new-http-offline-depot-support-for-vcf-installer-fleet-depot-service.html) and [side-loading VCF binaries for air-gapped environments](https://williamlam.com/2026/06/vcf-9-1-side-loading-vcf-binaries-into-vcf-installer-fleet-depot-service-for-air-gapped-environments.html).

### VCF Installer — initial fleet deploy  (token: `/v1/tokens` → `accessToken`)
This Installer endpoint is **current in 9.1** (not deprecated).
```bash
SM=https://vcf-installer.example.com
TOKEN=$(curl -sk -X POST $SM/v1/tokens -H 'Content-Type: application/json' \
  -d '{"username":"admin@local","password":"<INSTALLER_ADMIN_PASSWORD>"}' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["accessToken"])')

curl -sk -X PUT $SM/v1/system/settings/depot \
  -H "Authorization: Bearer $TOKEN" -H 'Content-Type: application/json' \
  -d '{"depotConfiguration":{"isOfflineDepot":true,"url":"http://depot.example.com:8888"}}'
```

### VCF Fleet Depot Service — Day-N, owned by VCF Operations  (token: `/api/v1/identity/token` → `access_token`)
The Fleet uses a **different token endpoint** from the Installer — mint a fresh token here, do **not** reuse the Installer's `$TOKEN`. Note the response field is `access_token` (snake_case), and the request is form-encoded.
```bash
MS=https://vcf-ops.example.com       # VCF Operations / Management Services FQDN (token host)
FLEET=https://fleet.example.com      # VCF Fleet FQDN (may be the same appliance)

FTOKEN=$(curl -sk -X POST $MS/api/v1/identity/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'grant_type=password' \
  --data-urlencode 'username=admin@local' \
  --data-urlencode 'password=<VCF_OPS_ADMIN_PASSWORD>' \
  | python3 -c 'import sys,json;print(json.load(sys.stdin)["access_token"])')

curl -sk -X PUT $FLEET/depot-service/api/depot/v1/connectivity \
  -H "Authorization: Bearer $FTOKEN" -H 'Content-Type: application/json' \
  -d '{"depotConfiguration":{"depotType":"OFFLINE","url":"http://depot.example.com:8888"}}'
```

> **Not SDDC Manager (changed in 9.1).** Depot configuration has moved to **VCF Operations → Fleet Management → Lifecycle → Binary Management**. The old SDDC Manager `PUT /v1/system/settings/depot` with an `offlineAccount` + `hostname`/`port` body is the **legacy VCF 5.2 / 9.0 schema** ([Broadcom Depot Settings API reference — tagged 5.2.x](https://developer.broadcom.com/xapis/vmware-cloud-foundation-api/latest/depot-settings/)) — don't use it for a 9.1 PoC.

> **HTTPS + basic-auth offline depot — verify before relying on it.** Every authoritative 9.1 example covers only the **HTTP, no-auth** (`"url":"http://…"`) case shown above. There is currently **no published 9.1 example** of how a username/password is supplied on the new `url`-style payload, so this guide deliberately does **not** invent one. If you need an authenticated HTTPS depot, confirm the exact payload against your build's Fleet Depot Service API reference (Swagger) first.

Then in **VCF Operations → Fleet Management → Lifecycle → Binary Management**, your downloaded
bundles appear and can be used for install/upgrade/patch.

> **HTTP scheme support:** On **VCF 9.1**, HTTP offline depots are supported **natively
> via the API** (`"url":"http://…"`) — set it through the API (not the UI) and serve HTTP on an HTTP
> port like `8888`, not 443. On **VCF 9.0** there was no native HTTP support; the workaround was the
> LCM property `lcm.depot.adapter.httpsEnabled=false` in
> `/opt/vmware/vcf/lcm/lcm-app/conf/application-prod.properties` (`systemctl restart lcm`) — but that
> property is **overwritten on upgrade** and **disables HTTPS globally** (breaking online depots)
> while set, so prefer the 9.1 API method. Tell‑tale of the bug: the depot URL shows `http://…:443`
> (HTTP scheme on an HTTPS port).

---

## 8. Quick reference — the whole flow

```bash
# 1. ID + activation code
./vcf-download-tool configuration generate --software-depot-id
#    register ID at vcf.broadcom.com → save code to /path/to/activation-code.txt

# 2. Download (install + upgrade + esx)
./vcf-download-tool binaries download --sku VCF --vcf-version <VCF_VERSION> \
  --depot-download-activation-code-file /path/to/activation-code.txt \
  --type INSTALL --automated-install --depot-store /path/to/depot
./vcf-download-tool binaries download --sku VCF --vcf-version <VCF_VERSION> \
  --depot-download-activation-code-file /path/to/activation-code.txt \
  --type UPGRADE --depot-store /path/to/depot
./vcf-download-tool esx download \
  --depot-download-activation-code-file /path/to/activation-code.txt --depot-store /path/to/depot

# 3. Serve /path/to/depot
sudo systemctl enable --now depot-server.service     # http://depot.example.com:8888

# 4. Point VCF at it (API — see Phase 5)
```

---

## 9. Sources

- **Broadcom** — [Download bundles to an offline depot (VCF 9.1)](https://techdocs.broadcom.com/us/en/vmware-cis/vcf/vcf-9-0-and-later/9-1/lifecycle-management/binary-management-for-vmware-cloud-foundation/download-bundles-to-an-offline-depot.html)
- **William Lam** — [VCF 9.1: New HTTP Offline Depot Support](https://williamlam.com/2026/05/vcf-9-1-new-http-offline-depot-support-for-vcf-installer-fleet-depot-service.html)
- **William Lam** — [VCF 9.1: VCF Download Tool (VCFDT) Cheatsheet](https://williamlam.com/2026/05/vcf-9-1-vcf-download-tool-vcfdt-cheatsheet.html)
- **William Lam** — [Host VCF Offline Depot with Python SimpleHTTPServer + Auth](https://williamlam.com/2025/01/quick-tip-easily-host-vmware-cloud-foundation-vcf-offline-depot-using-python-simplehttpserver-with-authentication.html)
- **William Lam** — [VCF Software Depot Structure Deep Dive](https://williamlam.com/2025/10/vcf-software-depot-structure-deep-dive-for-install-upgrade.html)
