# Home Assistant Secure External Access via Cloudflare Tunnel & mTLS

This document outlines the step-by-step configuration used to expose Home Assistant (`home-assistant-45ty78.jordangrossman.com`) securely to the internet without opening inbound ports, locked down using **Cloudflare Tunnel** and **Domain-Level mTLS Client Certificate Authentication** (without relying on WAF Custom Rules).

---

## 🏗️ Architecture Overview

```
[ Mobile Phone / Laptop ] 
       │ (pfx/p12 Client Cert)
       ▼
[ Cloudflare TLS Edge (Hosts Setting) ] ──(Dropped at TLS handshake if no Cert)
       │ (Verified mTLS Session)
       ▼
[ Cloudflare Tunnel (cloudflared) ]
       │ (Outbound Encrypted Tunnel)
       ▼
[ Home Assistant (HAOS) ]
```

- **Zero Inbound Open Ports:** The home router does not require any port forwarding.
- **Domain-Level mTLS Enforcement:** TLS handshake fails automatically at the Cloudflare edge for any request missing a valid client certificate.
- **Trusted Reverse Proxy Configuration:** Home Assistant accepts connections relayed through Cloudflare's subnets via the UI system settings.

---

## 🛠️ Step 1: Configure Home Assistant Reverse Proxy Settings

Modern Home Assistant versions manage reverse proxy and HTTP settings directly via the GUI.

1. In Home Assistant, navigate to **Settings > System > Network**.
2. Scroll down to the **HTTP Server** section.
3. Enable **Use X-Forwarded-For** (or *Use reverse proxy*).
4. Under **Trusted Proxies**, add the following Docker subnets:
   - `172.30.33.0/24`
   - `172.30.0.0/16`
5. Click **Save** and restart Home Assistant.

---

## 📦 Step 2: Install & Configure Cloudflared (Apps / Add-ons)

1. Navigate to **Settings > Apps** (or *Add-ons*).
2. Open the **App Store** / **Add-on Store**.
3. Click the three dots (top right) ➔ **Repositories**.
4. Add the repository:
   ```text
   https://github.com/brenner-tobias/ha-addons
   ```
5. Search for and install **Cloudflared**.
6. Open the **Cloudflared** configuration tab:
   - Set **External Hostname**: `home-assistant-45ty78.jordangrossman.com`
   - Save configuration.
7. Switch to the **Info** tab, enable **Watchdog** & **Auto-update**, and click **Start**.
8. View the **Log** tab, open the generated Cloudflare authentication URL (`https://dash.cloudflare.com/argotunnel?token=...`), and authorize your domain.

---

## 🔒 Step 3: Enable Host mTLS Enforcement & Generate Certificates

Instead of using WAF Rules, mTLS enforcement is handled directly at the SSL/TLS transport layer by assigning your hostname to the Client Certificates configuration.

### 3.1 Enable Host mTLS Enforcement
1. Log into [Cloudflare Dashboard](https://dash.cloudflare.com).
2. Select your domain (`jordangrossman.com`).
3. Navigate to **SSL/TLS > Client Certificates**.
4. Under **Hosts**, click **Edit**.
5. Add `home-assistant-45ty78.jordangrossman.com` to the list and click **Save**.
   *(Adding the host here forces Cloudflare to mandate and validate a valid client certificate during the initial TLS handshake).*

### 3.2 Create Client Certificate
1. On the same **Client Certificates** page, click **Create Certificate**.
2. Keep default settings (RSA 2048, 10-year validity).
3. Set Common Name to `home-assistant-45ty78.jordangrossman.com` (or a device descriptive name).
4. Save the returned **Certificate** (`cert.crt`) and **Private Key** (`key.key`) locally by copying and pasting into a text editor.

### 3.3 Convert to Mobile PKCS#12 Bundle (.pfx / .p12)
Run OpenSSL on your local machine to bundle the key pair:
```bash
openssl pkcs12 -export -out ha-client.pfx -inkey key.key -in cert.crt
```

> [!WARNING]
> Set a strong password when prompted during OpenSSL export. You will need this password when importing the certificate to your mobile device.

---

## 📱 Step 4: Install Certificate on Mobile / Client Devices

### iOS / iPadOS
1. Send `ha-client.pfx` to the device (AirDrop / Files).
2. Open the file in **Files** app ➔ Tap to load profile.
3. Go to **Settings > Profile Downloaded** ➔ Install.
4. Go to **Settings > General > About > Certificate Trust Settings** ➔ Enable full trust for root certificate.
5. In the **Home Assistant Companion App**, navigate to **Settings > Companion App > Server Settings** and select the installed Client Certificate.

### Android
1. Download `ha-client.pfx` to the device.
2. Go to **Settings > Security > Encryption & Credentials > Install a Certificate > User / VPN Certificate**.
3. Select the file and enter your export password.
4. Set up the certificate in the **Home Assistant Companion App** settings.

---

## 🧪 Step 5: Verification & Testing

Verify that domain-level mTLS enforcement is active using `curl`:

### Test 1: Unauthorized Device (Should return HTTP 403 / TLS handshake drop)
```bash
curl -s -o /dev/null -w "%{http_code}\n" https://home-assistant-45ty78.jordangrossman.com
# Expected Output: 403
```

### Test 2: Authorized Device with Cert (Should return HTTP 200 OK)
```bash
curl -s -o /dev/null -w "%{http_code}\n" --cert cert.crt --key key.key https://home-assistant-45ty78.jordangrossman.com
# Expected Output: 200
```

---

## 📌 Summary Details
- **Domain:** `home-assistant-45ty78.jordangrossman.com`
- **Tunnel Type:** Cloudflared HAOS Add-on (Brenner Repository)
- **mTLS Enforcement Method:** SSL/TLS Hosts Configuration (No WAF Add-on Required)
- **Created Date:** September 2026
