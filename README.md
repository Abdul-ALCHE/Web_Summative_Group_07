# Group 07 for Secure Web Infrastructure

A single-VM production-style deployment: two independent backend services sit behind
an NGINX reverse proxy that load-balances between them, terminates TLS on a
self-signed certificate, and is the only thing UFW lets the outside world talk to.
Built, broken, and fixed twice — on purpose the second time — on Parrot Security OS.

---

## What this repo demonstrates

Rather than one script doing everything, the system is split into layers the way a
real deployment would be: two disposable, independently-restartable backend
processes; a single reverse proxy that owns routing, load distribution and TLS; and a
firewall that enforces the boundary between "public" and "internal" regardless of
what the application layer does.

Two things separate this from a checklist exercise:

- **It broke for real once.** Midway through configuring NGINX, the main config file
  itself became invalid and the service refused to reload. That incident is
  documented as Fault #1 below, diagnosed the same way a deliberate fault would be.
- **A second fault was then introduced on purpose** — an NGINX `backup` directive
  quietly pulled one backend out of rotation — to prove the diagnostic method works
  even when nothing is obviously "down."

---

## Architecture

```
                         CLIENT
                           |
                 HTTP :80  | 301 redirect
                           v
                    +-------------+
                    |  UFW        |   allow 22, 80, 443
                    |  firewall   |   deny  3001, 3002
                    +-------------+
                           |
                      HTTPS :443
                           v
                 +--------------------+
                 |       NGINX        |
                 |  reverse proxy +   |
                 |  load balancer +   |
                 |  TLS termination   |
                 +--------------------+
                     /            \
                    v              v
        +------------------+   +------------------+
        |    backend1       |   |    backend2      |
        | 127.0.0.1:3001    |   | 127.0.0.1:3002   |
        |  "...Server 1"    |   |  "...Server 2"   |
        +------------------+   +------------------+
```

**Why it's laid out this way:** the backends never need to know a firewall or TLS
certificate exists — NGINX absorbs all of that. Losing one backend doesn't lose the
service, because NGINX is the only thing that knows there are two.

---

## Stack summary

| Layer | Technology | Listens on | Notes |
|---|---|---|---|
| Firewall | UFW | — | Default-deny inbound; explicit allow/deny list |
| Edge / proxy | NGINX | `80`, `443` | Reverse proxy, load balancer, TLS termination |
| TLS | OpenSSL (self-signed) | — | RSA 2048, 365-day cert, key restricted `chmod 600` |
| App tier | Python 3 `http.server` × 2 | `127.0.0.1:3001` / `3002` | Loopback-only, systemd-managed |
| Process supervision | systemd | — | `Restart=on-failure` on both backend units |

---

## Getting it running

```bash
# 1. Packages
sudo apt update && sudo apt install -y nginx ufw curl openssl python3 openssh-server

# 2. Backends (systemd units point at /opt/backend1/server.py and /opt/backend2/server.py)
sudo systemctl enable --now backend1.service
sudo systemctl enable --now backend2.service

# 3. NGINX (upstream + HTTPS server block — see NGINX/ below)
sudo nginx -t && sudo systemctl reload nginx

# 4. Firewall
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw deny 3001/tcp
sudo ufw deny 3002/tcp
sudo ufw enable
```

Sanity check everything is alive:

```bash
for i in {1..10}; do curl -sk https://127.0.0.1; done
```

Ten lines, a mix of `Server 1` and `Server 2` — that's the whole point of the load
balancer proven in one command.

---

## NGINX configuration

```nginx
upstream backend_pool {
    server 127.0.0.1:3001;
    server 127.0.0.1:3002;
}

server {
    listen 80;
    server_name localhost;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        proxy_pass http://backend_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Nothing is reloaded without validating it first:

```bash
sudo nginx -t          # must say "syntax is ok" / "test is successful"
sudo systemctl reload nginx
```

---

## TLS certificate

```bash
openssl req -x509 -nodes -newkey rsa:2048 -days 365 \
  -keyout /etc/nginx/ssl/nginx-selfsigned.key \
  -out    /etc/nginx/ssl/nginx-selfsigned.crt \
  -subj   "/C=MU/ST=PortLouis/L=PortLouis/O=WebInfrastructure/OU=Group_07/CN=localhost" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

sudo chmod 600 /etc/nginx/ssl/nginx-selfsigned.key
```

| File | Contents | Shareable? |
|---|---|---|
| `nginx-selfsigned.crt` | Public certificate | Yes |
| `nginx-selfsigned.key` | Private key | **No — never commit this** |

```bash
# redirect check
curl -i http://127.0.0.1        # expect 301 -> https://127.0.0.1/

# certificate check
openssl x509 -in /etc/nginx/ssl/nginx-selfsigned.crt -noout -subject -issuer -dates
```

---

## Firewall policy

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow 22/tcp  comment 'SSH administration'
sudo ufw allow 80/tcp  comment 'HTTP redirect to HTTPS'
sudo ufw allow 443/tcp comment 'HTTPS web service'
sudo ufw deny  3001/tcp comment 'Backend 1 internal only'
sudo ufw deny  3002/tcp comment 'Backend 2 internal only'
```

| Port | Direction | Rule | Why |
|---|---|---|---|
| 22 | in | allow | Admin access — allowed *before* `ufw enable` so SSH sessions aren't dropped |
| 80 | in | allow | Needs to be reachable just long enough to issue the 301 redirect |
| 443 | in | allow | The only port the application is actually served on |
| 3001 / 3002 | in | **deny** | Backends already bind to loopback; the firewall is a second, independent lock on the same door |

---

## Fault log

Two incidents, documented the same way regardless of whether they were planned.

### Fault #1 — NGINX wouldn't reload (found by accident)

| | |
|---|---|
| **Symptom** | `nginx -t` failed with `emerg ... No such file or directory in /etc/nginx/nginx.conf:1`; every reload attempt failed. |
| **Evidence** | The error pointed at the *main* config, not the site file — `sites-available`/`sites-enabled` were already correct. |
| **Investigation** | `cat /etc/nginx/nginx.conf` to walk the `events`/`http` blocks; a routine `apt upgrade` ruled out a package regression. |
| **Cause** | The main config's top-level structure was broken, so NGINX couldn't parse *any* config, site file or otherwise. |
| **Resolution** | Rebuilt `nginx.conf` in `nano`; `nginx -t` passed; reload succeeded; service confirmed `active (running)`. |

### Fault #2 — one backend silently stopped taking traffic (planted)

```nginx
server 127.0.0.1:3002 backup;   /* added deliberately */
```

| | |
|---|---|
| **Symptom** | Ten requests in a row, every single one `Server 1`. No errors, no crash — just no Server 2. |
| **Evidence** | `backend2.service` was `active (running)`, port 3002 was listening, and a direct `curl` to it worked fine. The *running* NGINX config (not just the file) showed `backup` on that line. |
| **Investigation** | Stopped `backend1.service` — Server 2 responded instantly. That behaviour only makes sense if Server 2 was a standby, not a full pool member. |
| **Cause** | `backup` tells NGINX "only use this server if every non-backup server is down" — which is exactly what was observed. |
| **Resolution** | Deleted `backup`, revalidated, reloaded. Ten requests afterward: a healthy mix of Server 1 and Server 2 again. |

---

## Evidence

Terminal captures from the actual Parrot OS session, one per milestone above. The
complete set (20 screenshots) lives in [`images/`](images/) — a subset is shown here.

| | |
|---|---|
|  Both backend services `active (running)`, each listening only on its own loopback port. |
| Ten requests through NGINX, alternating Server 1 / Server 2 — load balancing, proven. |
| HTTP → 301 → HTTPS, and the HTTPS response itself, side by side. |
| Final UFW ruleset: 22/80/443 allowed, 3001/3002 denied. |
| After Fault #2 was fixed — balanced traffic restored, all three services healthy. |

---

## Repository layout

```
Group_07_SecureWebInfrastructure/
├── README.md
├── Technical_Report.pdf
├── Demo/demo_link.txt
├── Architecture/architecture_diagram.png
├── NGINX/nginx_site_configuration.txt
├── Backend/
│   ├── backend1/{server.py, backend1.service}
│   └── backend2/{server.py, backend2.service}
├── TLS/
│   ├── certificate_information.txt
│   ├── public_certificate_if_used.crt
│   └── NEVER_INCLUDE_PRIVATE_KEY.txt
├── Firewall/firewall_rules.txt
└── images/
    └── shot-01.png … shot-20.png
---

## Team

| Member | Owned | Also demos |
|---|---|---|
| Grace| Backend services, environment setup | Ports & systemd units |
| Jacinha| NGINX config, load-balancing proof | Reverse proxy walkthrough |
| Nwando | TLS certificate, HTTPS, redirect | Certificate inspection |
| Abduljabar | UFW rules, both fault investigations | Live troubleshooting |

---

## AI use

Claude was used to help structure this documentation and interpret NGINX/UFW/OpenSSL
output while writing it up. All commands, configuration, and the two fault
investigations above were run and verified by the group on its own machine.

---
