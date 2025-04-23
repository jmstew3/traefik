# Cloudflare ⇄ Traefik ⇄ NAS — End‑to‑End TLS Setup Guide
**Purpose**  
You want to serve a domain (e.g. `example.com`) from your NAS at home, with:
* Cloudflare acting as the public edge (orange‑proxied DNS record)
* A Cloudflare **Origin CA certificate** securing the hop between Cloudflare and your network
* Traefik v2 running in a Docker Compose stack on the NAS as the reverse‑proxy in front of your apps

This file gives step‑by‑step instructions, plus the reasoning behind every task, so you can copy it anywhere or commit it to your infra repo.

---

## 1 · Prerequisites
| Requirement | Why it matters |
|-------------|----------------|
| Domain added to Cloudflare & proxied (orange cloud) | Origin CA certs are trusted only by Cloudflare |
| Admin access to your router / firewall | To forward TCP 443 (and optionally 80) to the NAS |
| NAS with Docker Compose & Traefik v2.x | Traefik will present the Origin cert to Cloudflare |
| SSH / terminal access to the NAS host OS | To create folders, copy certs, and launch Compose |

---

## 2 · Issue the Origin CA certificate in Cloudflare
1. **Cloudflare Dashboard → SSL/TLS → Origin Server → *Create Certificate***  
   *Hostnames* → `example.com`, `*.example.com`  
   *Key type* → RSA 2048 (or ECC P‑256)  
   *Validity* → leave at 15 years unless you prefer shorter.  
2. Click **Create** → copy / download:  
   * `origin.pem`  — **certificate**  
   * `origin-key.pem` — **private key** (keep secret)  
3. (Optional) download `origin_ca_rsa_root.pem` — Cloudflare’s Origin CA root.  
   (Traefik doesn’t need it, but keep it for cURL testing.)

**Why?**  
Browser‑trusted CAs can’t issue certs for private IPs. Cloudflare Origin CA solves that by creating a cert chain that **only Cloudflare trusts**, perfect when all traffic enters through Cloudflare anyway.

---

## 3 · Prepare directories on the NAS

```bash
sudo mkdir -p /opt/traefik/{certs,dynamic}
sudo chmod 700 /opt/traefik/certs
# copy the files you saved locally
sudo cp origin.pem origin-key.pem /opt/traefik/certs/
```

### Dynamic TLS definition (`/opt/traefik/dynamic/tls.yml`)
```yaml
tls:
  stores:
    default:
      defaultCertificate:
        certFile: /certs/origin.pem
        keyFile:  /certs/origin-key.pem
  certificates:
    - certFile: /certs/origin.pem
      keyFile:  /certs/origin-key.pem
```
*Reason:* Declares the Origin cert as the **fallback** for every HTTPS router unless another resolver is specified.

---

## 4 · Docker Compose for Traefik (`docker-compose.yml`)
```yaml
version: "3.8"

services:
  traefik:
    image: traefik:v2.11
    container_name: traefik
    restart: unless-stopped

    command:
      - "--providers.docker=true"
      - "--providers.file.directory=/dynamic"
      - "--providers.file.watch=true"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"

    ports:
      - "80:80"
      - "443:443"

    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - /opt/traefik/dynamic:/dynamic:ro
      - /opt/traefik/certs:/certs:ro
```

Bring it up:
```bash
docker compose up -d
```

---

## 5 · Label your app containers
```yaml
labels:
  - "traefik.http.routers.myapp.rule=Host(`example.com`)"
  - "traefik.http.routers.myapp.entrypoints=websecure"
  - "traefik.http.routers.myapp.tls=true"
```
Traefik now presents the Origin cert for any request on `websecure`.

---

## 6 · Router / Firewall rules

| Port | To | Protocol | Purpose |
|------|----|----------|---------|
| **443** | `<NAS-IP>:443` | TCP | Required for Cloudflare → Traefik TLS |
| 80 (optional) | `<NAS-IP>:80` | TCP | HTTP → HTTPS redirect / future ACME HTTP‑01 |

**Tip:** restrict 80/443 to Cloudflare IP ranges to stop direct hits on your home IP.

---

## 7 · Cloudflare encryption mode

`Dashboard → SSL/TLS → Overview → Encryption Mode → **Full (strict)**`

Full (strict) forces Cloudflare to verify the Origin cert and blocks plaintext.

---

## 8 · Testing

```bash
# From a machine outside your LAN
curl -I https://example.com

# Optional: hit the NAS directly to inspect the chain
curl --resolve example.com:443:<YOUR_HOME_IP>          --cacert origin_ca_rsa_root.pem -I https://example.com
```

Expect `HTTP/1.1 200 OK` and a TLS handshake with **CN=example.com**.

---

## 9 · Troubleshooting

| Symptom | Likely cause |
|---------|-------------|
| **525 Handshake Failed** | Wrong cert/key, port 443 not forwarded, or Cloudflare not set to *Full (strict)* |
| Browser shows Cloudflare’s universal cert | You’re bypassing Cloudflare (grey‑cloud DNS) |
| Traefik log: *“No default certificate”* | Path typo or volume mis‑mount — make sure `/certs/origin.pem` exists in the container |

---

## 10 · Alternative: Cloudflare Tunnel

If you cannot open inbound ports, install `cloudflared` on the NAS and create a Tunnel; Traefik listens on `localhost:8000`, `cloudflared` connects outbound to Cloudflare. No NAT rules required, but keep the Origin cert so the tunnel connection still uses mutual TLS.

---

**Result:**  
Cloudflare ↔︎ (TLS) ↔︎ Traefik ↔︎ your Docker services, with the Origin CA certificate enforcing end‑to‑end encryption while keeping your true IP private.
