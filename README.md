# Infrastructure Design Document (IDD) — peviitor.ro

## Version 1.1 — August 2026

---

## 1. Introducere

### 1.1 Scop
Acest document descrie infrastructura hardware si software a platformei **peviitor.ro** — un motor de cautare open-source a locurilor de munca din Romania. Documentul acopera topologia retelei, configuratia serverelor, arhitectura de deployment, strategia de backup si recuperare in caz de dezastru.

### 1.2 Documente de referinta
| Document | URL |
|---|---|
| Software Architecture Design (SAD) | https://sad.peviitor.ro |
| Test Strategy | https://github.com/peviitor-ro/test_strategy_peviitor-ro |
| Onboarding Portal | https://onboarding.peviitor.ro |

---

## 2. Topologia Infrastructurii

```
+------------------------------------------------------------------+
|                      RCS&RDS (ISP)                               |
|                     IP Dinamic 86.122.35.88                       |
|                     Fibra Optica (1 Gbps)                         |
+--------------------------------+---------------------------------+
                                 |
                         +-------v--------+
                         |    ONT (ONU)   |
                         |   Fibra -> Ethernet |
                         +-------+--------+
                                 |
                         +-------v--------+
                         |   Router WiFi 6 |
                         |   192.168.1.1   |
                         |   NAT, Port Fwd |
                         +-------+--------+
                                 |
                         +-------v--------+
                         | Switch Gigabit  |
                         |   1 Gbps       |
                         +-------+--------+
                                 |
            +--------------------+--------------------+
            |                    |                    |
    +-------v--------+   +-------v--------+   +------v--------+
    | RPi 5 (16GB)   |   | RPi 5 (4GB)    |   | RPi 4         |
    | SOLR Server    |   | API Server     |   | TEST Server   |
    | Productie      |   | Productie      |   | test.peviitor.ro|
    | 192.168.1.134  |   | 192.168.1.135  |   | 192.168.1.142 |
    |                |   |                |   |               |
    | Docker:        |   | Docker:        |   | Docker:       |
    | solr:10-slim   |   | peviitor-api   |   | peviitor-api  |
    | :8983          |   | orase-api      |   | (Apache PHP)  |
    |                |   | nginx-proxy-mgr|   | peviitor-solr |
    |                |   | :80/443/81     |   | :8983         |
    |                |   |                |   | OpenResty     |
    |                |   |                |   | :80/443       |
    +----------------+   +----------------+   +---------------+

                    +-------------------------+
                    |   GitHub Pages          |
                    |   Frontend (React)      |
                    |   peviitor.ro           |
                    +-------------------------+

                    +-------------------------+
                    |   GitHub Actions        |
                    |   Scrapers (cron)       |
                    |   Python/Node/JMeter    |
                    +-------------------------+

                    +-------------------------+
                    |   CloudFlare            |
                    |   CDN, DNS, SSL, DDoS   |
                    +-------------------------+
```

### 2.1 Flux trafic

```
Utilizator (Browser)
    |
    | HTTPS (peviitor.ro)
    v
CloudFlare (CDN, SSL, cache)
    |
    | DNS: zimbor.go.ro
    v
RCS&RDS (fibra optica 1 Gbps)
    |
    v
ONT (fibra → Ethernet)
    |
    v
Router WiFi 6 (NAT, port forwarding :80/:443)
    |
    v
Switch Gigabit (1 Gbps)
    |
    v
RPi API (Nginx Proxy Manager)
    |
    | :8080 (PHP BFF)
    v
RPi SOLR (HTTP LAN :8983)
    |
    v
Index SOLR (job / company)
```

---

## 3. Server API — Raspberry Pi 5 (4GB)

### 3.1 Hardware

| Componenta | Detalii |
|---|---|
| **Model** | Raspberry Pi 5 Model B Rev 1.0 |
| **SoC** | Broadcom BCM2712 (Cortex-A76, 4 nuclee, 64-bit) |
| **CPU** | 4 nuclee @ ~2.4 GHz |
| **RAM** | 4 GB LPDDR4X |
| **Stocare** | MicroSD 59.4 GB (25 GB folositi, 31 GB liberi) |
| **GPU** | VideoCore VII |
| **Retea** | Ethernet 1000 Mbps (eth0: 192.168.1.135/24) |
| **Temperatura** | ~49.9 °C |

### 3.2 Software

| Componenta | Detalii |
|---|---|
| **OS** | Debian 12 (Bookworm) |
| **Kernel** | Linux 6.12.87+rpt-rpi-2712 (aarch64) |
| **Hostname** | `api` |
| **Uptime** | 4 zile (la ultimul restart) |
| **Window Manager** | labwc (Wayland) |
| **Display Manager** | LightDM |
| **VNC** | wayvnc (rpi-connect) |

### 3.3 Docker Containers

| Nume | Imagine | Porturi | Rol |
|---|---|---|---|
| peviitor-api | php:8.3-apache | 8080 → 80 | API BFF principal |
| orase-api | php:8.3-apache | 8081 → 80 | API orase.peviitor.ro |
| npm-app | jc21/nginx-proxy-manager | 80, 81, 443 | Reverse proxy, SSL |

### 3.4 Servicii systemd

| Serviciu | Rol |
|---|---|
| `docker.service` | Container runtime |
| `netdata.service` | Monitoring infrastructura |
| `ssh.service` | Acces remote securizat |
| `solr-monitor.service` | Monitor SOLR (Discord webhook) |
| `avahi-daemon.service` | mDNS (descoperire retea locala) |
| `cron.service` | Task-uri programate |

### 3.5 Tooling si Runtime-uri

| Tool | Versiune |
|---|---|
| Node.js | v20.20.2 |
| npm | 10.8.2 |
| Python | 3.11.2 |
| GCC | 12 |
| Git | instalat |
| Make | instalat |
| Docker Engine | activ |
| Docker Compose | activ |

---

## 4. Server SOLR — Raspberry Pi 5 (16GB)

### 4.1 Hardware

| Componenta | Detalii |
|---|---|
| **Model** | Raspberry Pi 5 Model B Rev 1.1 |
| **CPU** | ARM Cortex-A76, 4 nuclee @ 2.4 GHz (max) / 1.5 GHz (min) |
| **Cache** | L1d 256 KiB, L1i 256 KiB, L2 2 MiB, L3 2 MiB |
| **RAM** | 16 GB LPDDR4X (15 GiB usable) |
| **Swap** | 16 GiB swapfile + 2 GiB zram (zstd) |
| **Stocare** | microSD 64 GB (33 GB folositi) |
| **Retea** | Ethernet 1000 Mbps (eth0: 192.168.1.134/24) |

### 4.2 Software

| Componenta | Detalii |
|---|---|
| **OS** | Debian 13 (Trixie), v13.5 |
| **Kernel** | 6.18.29+rpt-rpi-2712 #1 SMP PREEMPT (aarch64) |
| **Hostname** | `solr-pi` |
| **Uptime** | 4 zile |

### 4.3 Docker Containers

| Nume | Imagine | Porturi | Rol |
|---|---|---|---|
| solr-container | solr:10-slim | 0.0.0.0:8983 → 8983 | Apache SOLR search engine |

**Imagini stocate:** solr:latest (1.25 GB), solr:9-slim (531 MB), solr:10-slim (675 MB), alpine, jq

### 4.4 Runtime-uri

| Runtime | Versiune |
|---|---|
| Python | 3.13.5 |
| Node.js | v24.16.0 |
| npm | 11.13.0 |
| GCC/G++ | 14.2.0 |
| Git | 2.47.3 |
| OpenSSL | 3.5.6 |
| Docker CE | 29.5.2 |
| Docker Compose | 5.1.4 |

---

## 5. Retea

### 5.1 Topologie retea locala

| Dispozitiv | IP | Interfata | Rol |
|---|---|---|---|
| ONT (ONU) | — | Fibra optica → Ethernet | Convertor semnal optic |
| Router WiFi 6 | 192.168.1.1 | WAN: ONT / LAN: Switch Gigabit | NAT, port forwarding, Wi-Fi |
| Switch Gigabit | — | 4 porturi 1 Gbps | Conectare RPi-uri la retea |
| RPi 5 SOLR | 192.168.1.134 | eth0 → Switch | SOLR productie (16GB) |
| RPi 5 API | 192.168.1.135 | eth0 → Switch | API productie (4GB) |
| RPi 4 TEST | 192.168.1.142 | eth0 → Switch | Mediu test (test.peviitor.ro) |
| RPi SOLR (docker) | 172.17.0.1/16 | docker0 | Retea interna containere |

### 5.2 Acces extern

| Serviciu | URL | Metoda |
|---|---|---|
| Frontend | https://peviitor.ro | GitHub Pages + CloudFlare |
| API BFF | https://api.peviitor.ro | DDNS + Nginx Proxy Manager |
| SOLR public | https://solr.peviitor.ro | Prin API (indirect) |
| SOLR admin | https://solr.peviitor.ro | Basic Auth (direct) |
| Test frontend | https://test.peviitor.ro | RPi 4 — OpenResty |
| Test API | https://test.peviitor.ro/swagger-ui | RPi 4 — Apache PHP 8.2 |
| Test SOLR | https://testsolr.peviitor.ro | RPi 4 — SOLR 10 (Basic Auth) |

### 5.3 DDNS

- **Furnizor:** go.ro
- **Hostname:** zimbor.go.ro
- **IP actual:** 86.122.35.88
- **ISP:** RCS&RDS (IP dinamic) — DDNS configurat direct la nivel ISP
- **TTL:** Scazut (actualizare rapida la schimbare IP)
- **Conexiune:** Fibra optica → ONT (ONU) → Ethernet → Router WiFi 6
- **Viteza conexiune:** 1 Gbps (furnizata de ISP)
- **Port forwarding pe router (WiFi 6):**
  - `:80` → `192.168.1.135:80`
  - `:443` → `192.168.1.135:443`

### 5.4 Domeniul peviitor.ro

| Detaliu | Valoare |
|---|---|
| **Domeniu** | peviitor.ro |
| **Data inregistrarii** | 07-04-2021 |
| **Registrar** | Claus Web SRL (doar achizitie, fara hosting) |
| **Deținător** | Persoană Fizică (GDPR) |
| **Servere nume** | maria.ns.cloudflare.com, razvan.ns.cloudflare.com |
| **Stare** | OK |

### 5.5 CloudFlare

- **DNS:** Nameservere: `maria.ns.cloudflare.com`, `razvan.ns.cloudflare.com`
- **CDN:** Proxy activ (orange cloud) pentru domeniile marcate `cf-proxied:true`
- **SSL:** Full (strict) — certificat CloudFlare
- **DDoS Protection:** Activ
- **Caching:** Active pentru assets statice

#### 5.5.1 Înregistrări DNS (export @ 2026-06-04)

**A Records (frontend GitHub Pages):**
| Nume | TTL | IP | Proxy |
|---|---|---|---|
| peviitor.ro | 1 | 185.199.108.153 | Nu |
| peviitor.ro | 1 | 185.199.109.153 | Nu |
| peviitor.ro | 1 | 185.199.110.153 | Nu |
| peviitor.ro | 1 | 185.199.111.153 | Nu |

**A Records (direct — fără proxy):**
| Nume | TTL | IP | Proxy |
|---|---|---|---|
| beta.peviitor.ro | 1 | 93.113.55.230 | Nu |
| ftp.peviitor.ro | 1 | 93.113.55.230 | Nu |
| mail.peviitor.ro | 1 | 93.113.55.230 | Nu |

**CNAME Records (către servicii externe):**
| Nume | TTL | Target | Proxy |
|---|---|---|---|
| admin.peviitor.ro | 1 | adminpeviitor.netlify.app | **Da** |
| apidoc.peviitor.ro | 1 | ghs.googlehosted.com | Nu |
| index.peviitor.ro | 1 | ghs.googlehosted.com | Nu |
| legal.peviitor.ro | 1 | ghs.googlehosted.com | Nu |
| onboarding.peviitor.ro | 1 | ghs.googlehosted.com | Nu |
| qa.peviitor.ro | 1 | ghs.googlehosted.com | Nu |
| romania.peviitor.ro | 1 | peviitor-ro.github.io | Nu |
| sad.peviitor.ro | 1 | peviitor-ro.github.io | Nu |
| scraper.peviitor.ro | 1 | ghs.googlehosted.com | Nu |
| scrapers.peviitor.ro | 1 | scraper-ui.netlify.app | **Da** |
| splash.peviitor.ro | 1 | ghs.googlehosted.com | Nu |
| v01.peviitor.ro | 1 | peviitor-ro.github.io | Nu |
| v02.peviitor.ro | 1 | peviitor-ro.github.io | Nu |
| www.peviitor.ro | 1 | peviitor-ro.github.io | Nu |

**CNAME Records (către DDNS zimbor.go.ro — infrastructura locală):**
| Nume | TTL | Target | Proxy |
|---|---|---|---|
| api.peviitor.ro | 1 | zimbor.go.ro | **Da** |
| netdata.peviitor.ro | 1 | zimbor.go.ro | **Da** |
| orase.peviitor.ro | 1 | zimbor.go.ro | Nu |
| pi4.peviitor.ro | 1 | zimbor.go.ro | **Da** |
| pi5.peviitor.ro | 1 | zimbor.go.ro | **Da** |
| sebi.peviitor.ro | 1 | zimbor.go.ro | **Da** |
| solr.peviitor.ro | 1 | zimbor.go.ro | Nu |
| test.peviitor.ro | 1 | zimbor.go.ro | Nu |
| testsolr.peviitor.ro | 1 | zimbor.go.ro | Nu |
| solrcluj.peviitor.ro | 1 | peviitor.go.ro | **Da** |

**MX Records:**
| Prioritate | Target |
|---|---|
| 25 | route1.mx.cloudflare.net |
| 61 | route3.mx.cloudflare.net |
| 96 | route2.mx.cloudflare.net |

**TXT Records:** DKIM, SPF, Google Site Verification, Discord verification.

---

## 6. Arhitectura de Deployment

### 6.1 Componente si unde ruleaza

| Componenta | Unde ruleaza | URL |
|---|---|---|
| Frontend (React) | GitHub Pages (cdn) | https://peviitor.ro |
| API BFF (PHP) | RPi API — Docker | https://api.peviitor.ro |
| API Orase (PHP) | RPi API — Docker | https://orase.peviitor.ro |
| SOLR Search | RPi SOLR — Docker | https://solr.peviitor.ro |
| Validator (Admin) | GitHub Pages | https://admin.peviitor.ro |
| Scrapers | GitHub Actions | — |

### 6.2 Nginx Proxy Manager

- **Imagine:** `jc21/nginx-proxy-manager`
- **Porturi:** `:80` (HTTP → redirect HTTPS), `:81` (Admin UI), `:443` (HTTPS)
- **Rol:** Reverse proxy principal — primește toate call-urile pe porturile 80/443 de la router și le direcționează către containerul/serviciul corespunzător în funcție de domeniu
- **SSL:** Toate domeniile configurate în NPM au certificate **Let's Encrypt** — NPM face SSL termination și redirect HTTP → HTTPS
- **Domenii configurate în NPM:**
  - `api.peviitor.ro` → `peviitor-api:80` (API BFF)
  - `orase.peviitor.ro` → `orase-api:80` (API orase)
  - `solr.peviitor.ro` → `192.168.1.134:8983` (cu Basic Auth)
  - `test.peviitor.ro` → RPi 4 TEST (frontend)
  - `testsolr.peviitor.ro` → RPi 4 TEST (SOLR test)
  - Alte domenii (admin, etc.) după configurare

### 6.3 Pipeline CI/CD

```
Developer commit → GitHub → GitHub Actions
                                |
                    +-----------+-----------+
                    |           |           |
                    v           v           v
                Lint        Test        Build
                    |           |           |
                    +-----------+-----------+
                                |
                    +-----------+-----------+
                    |           |           |
                    v           v           v
              Frontend      API         SOLR config
              (gh-pages)    (Docker)    (manual)
```

**Frontend:** Deploy automat pe GitHub Pages la merge pe `main`
**API:** Deploy automat pe RPi API la merge pe `master`
**SOLR:** Configuratie manuala (prin SSH sau scripturi)

---

## 7. Servicii si Monitorizare

### 7.1 Monitoring

| Instrument | Ce monitorizeaza | URL |
|---|---|---|
| Netdata | RPi API (CPU, RAM, disk, retea, temperatura) | Port 19999 (LAN) |
| Sentry | Erori frontend + API | https://sentry.io |
| CloudFlare Analytics | Trafic, cache, securitate | CloudFlare Dashboard |
| Microsoft Clarity | Comportament utilizatori (frontend) | https://clarity.microsoft.com |

### 7.2 Servicii active (RPi API)

| Serviciu | Status | Port |
|---|---|---|
| Docker Engine | activ | socket unix |
| Netdata | activ | 19999 |
| SSH | activ | 22 |
| Nginx Proxy Manager | activ | 80/443/81 |
| PHP API (peviitor-api) | activ | 8080 |
| PHP API (orase-api) | activ | 8081 |
| solr-monitor | activ | — (Discord webhook) |
| cron | activ | — |
| Raspberry Pi Connect | activ | wayvnc (acces remote prin browser) |

### 7.3 Servicii active (RPi SOLR)

| Serviciu | Status | Port |
|---|---|---|
| Docker Engine | activ | socket unix |
| SSH | activ | 22 |
| SOLR (solr-container) | activ | 8983 |
| Raspberry Pi Connect | activ | wayvnc (acces remote prin browser) |

### 7.4 Servicii active (RPi TEST)

| Serviciu | Status | Port |
|---|---|---|
| Docker Engine | activ | socket unix |
| OpenResty | activ | 80/443 |
| Apache PHP 8.2 (peviitor-api) | activ | 8081 |
| SOLR (peviitor-solr) | activ | 8983 |
| Raspberry Pi Connect | activ | wayvnc (acces remote prin browser) |

---

## 8. Backup si Disaster Recovery

### 8.1 Strategie backup

| Componenta | Frecventa | Metoda | Retentie |
|---|---|---|---|
| Index SOLR | Zilnic (02:00) | Script shell + snapshot | 7 zile |
| Configuratie Docker | Manual (la modificare) | Git commit | Permanent |
| API source code | Continuu | GitHub | Permanent |
| Frontend source code | Continuu | GitHub | Permanent |
| Nginx Proxy Manager | Manual (la modificare) | Export UI | — |

### 8.2 Procedura de restore SOLR

```bash
# 1. Oprire container SOLR
docker stop solr-container

# 2. Restaurare snapshot din backup
# (script specific in peviitor_core)

# 3. Pornire container SOLR
docker start solr-container

# 4. Verificare integritate index
curl -u user:pass "http://localhost:8983/solr/job/select?q=*:*&rows=1"
```

### 8.3 Riscuri si mitigari

| Risc | Impact | Mitigare |
|---|---|---|
| Defectiune microSD | Pierdere date | Backup zilnic; inlocuire rapida |
| IP dinamic RCS&RDS | Indisponibilitate externa | DDNS cu TTL scazut; fallback manual |
| Single point of failure (RPi) | API sau SOLR indisponibil | Scripturi de restore rapide |
| Fara replica SOLR | Pierdere index | Rebuild din scrapers + backup |
| Incendiu/furt la domiciliu | Pierdere totala hardware | Backup in cloud (GitHub) |

---

## 9. Securitate

### 9.1 Masuri de securitate

| Masura | Detalii |
|---|---|
| **SOLR Basic Auth** | security.json activat pe container |
| **CORS restrictionat** | API accepta doar domenii cunoscute |
| **CloudFlare WAF** | Protectie DDoS si OWASP Top 10 |
| **SSL/TLS** | Let's Encrypt + CloudFlare Full (strict) |
| **SSH** | Autentificare prin cheie (fara parola) — port 22 deschis doar in LAN |
| **Raspberry Pi Connect** | Administrare remote prin browser (connect.raspberrypi.com) — wayvnc |
| **Firewall** | iptables pe toate RPi-urile |
| **API Keys** | Stocate in environment variables (docker) |
| **GitHub Secret Scanning** | Detectare automată credentiale leakate |
| **GitHub Code Scanning (CodeQL)** | Analiza vulnerabilitati CWE per push |
| **Dependabot** | Update-uri automate dependinte vulnerabile |

### 9.2 Network Segmentation

- **Retea LAN** (192.168.1.0/24) — trustata
- **SOLR** accesibil doar din LAN (nu expus direct public)
- **API** expus public prin Nginx Proxy Manager
- **Frontend** pe GitHub Pages (CDN extern, complet izolat)

---

## 10. Specificatii SOLR

### 10.1 Container

| Parametru | Valoare |
|---|---|
| Imagine | solr:10-slim |
| Port | 8983 (TCP) |
| Volum | `/var/solr` (date index) |
| Autentificare | Basic Auth (security.json) |
| Network | bridge (172.17.0.0/16) |

### 10.2 Core-uri

| Core | Unique Key | Documente | Scop |
|---|---|---|---|
| `job` | `url` | ~40,000+ | Locuri de munca |
| `company` | `id` (cif) | ~1,000+ | Companii |

### 10.3 Backup SOLR

```bash
# Backup
curl "http://localhost:8983/solr/job/replication?command=backup&name=backup_$(date +%Y%m%d)"

# Restore
curl "http://localhost:8983/solr/job/replication?command=restore&name=backup_20260601"
```

---

## 11. Inventar Hardware

### 11.1 RPi API (4GB)

| Element | Valoare |
|---|---|
| Model | RPi 5 Model B Rev 1.0 |
| RAM | 4 GB LPDDR4X |
| Stocare | microSD 59.4 GB |
| IP | 192.168.1.135/24 |
| OS | Debian 12 (Bookworm) |
| Rol | API BFF + Nginx Proxy Manager |

### 11.2 RPi SOLR (16GB)

| Element | Valoare |
|---|---|
| Model | RPi 5 Model B Rev 1.1 |
| RAM | 16 GB LPDDR4X |
| Swap | 16 GiB swap + 2 GiB zram |
| Stocare | microSD 64 GB |
| IP | 192.168.1.134/24 |
| OS | Debian 13 (Trixie) |
| Rol | Apache SOLR 10.x |

### 11.3 RPi TEST — test.peviitor.ro

| Element | Valoare |
|---|---|
| **Model** | Raspberry Pi 4 (ARM Cortex-A72, 4 nuclee) |
| **RAM** | 8 GB (7.6 GiB usable) |
| **Swap** | 4 GB swap |
| **Stocare** | microSD 64 GB (58 GB utilizabili, ~41 GB liberi) |
| **IP Local** | 192.168.1.142/24 |
| **IP Public** | 86.122.35.88 (prin NAT, port forwarding) |
| **Hostname DNS inversat** | 86-122-35-88.rdsnet.ro |
| **OS** | Debian 13 (Trixie), kernel 6.18.39+rpt-rpi-v8 (aarch64) |
| **Reverse Proxy** | OpenResty (nginx + LuaJIT) — port 80 (→301 HTTPS) / 443 |
| **TLS** | Let's Encrypt (ECDSA P-384, TLS 1.3, AES-256-GCM) |
| **Docker Containers** | `peviitor-api` (Apache PHP 8.2, port 8081), `peviitor-solr` (Solr 10.0.0, port 8983) |
| **Frontend** | React SPA (search-engine, build mode `qa`) servit de Apache |
| **API** | PHP BFF v0/v1 + Swagger UI |
| **SOLR** | Core-uri `job` + `company` (subset ~5k joburi) |
| **Rol** | Mediu de test (test.peviitor.ro, testsolr.peviitor.ro) |
| **Conexiune** | Ethernet → Switch Gigabit |
| **SSH** | Port 22 inchis public — administrare doar din LAN sau consola fizica |

> **Istoric:** La 2026-08-09 s-a efectuat un upgrade hardware — RPi 4 (2 GB) → RPi 4 (8 GB), păstrându-se cardul microSD. RAM: 1.8 GB usable → 8 GB (7.6 GiB usable); adăugat swap 4 GB; IP local: 192.168.1.141 → 192.168.1.142.

### 11.4 Retea

| Element | Detalii |
|---|---|
| ISP | RCS&RDS — fibra optica, 1 Gbps |
| ONT (ONU) | Convertor fibra optica → Ethernet |
| Router | WiFi 6 (192.168.1.1) — NAT, port forwarding, acces wireless |
| Switch | Gigabit Ethernet (4 porturi) — conectează cele 3 RPi-uri |
| IP Public | 86.122.35.88 (dinamic) |
| DDNS | zimbor.go.ro (configurat la nivel ISP) |
| Conexiune | 1000 Mbps (toate RPi-urile, prin Ethernet) |

---

## 12. Medii

| Mediu | Frontend | API | SOLR | Date |
|---|---|---|---|---|
| **Productie** | https://peviitor.ro | https://api.peviitor.ro | https://solr.peviitor.ro | Full (~40k joburi) |
| **Test** | https://test.peviitor.ro | https://test.peviitor.ro/swagger-ui | https://testsolr.peviitor.ro | Subset (~5k joburi) |
| **Local** | localhost:3000 | localhost/api | Docker local | Minimal (seed) |

---

## 13. Glosar

| Termen | Definitie |
|---|---|
| **BFF** | Backend for Frontend — API care serveste specific clientului frontend |
| **DDNS** | Dynamic DNS — serviciu care actualizeaza DNS-ul pentru IP-uri dinamice |
| **NPM** | Nginx Proxy Manager — reverse proxy cu UI web |
| **RPi** | Raspberry Pi — computer single-board |
| **zram** | RAM comprimat folosit ca swap (performanta mai buna ca disk swap) |
| **WAF** | Web Application Firewall — protectie trafic web |
| **CORS** | Cross-Origin Resource Sharing — restrictie acces intre domenii |
| **CodeQL** | Motor de analiza semantica GitHub pentru vulnerabilitati |
| **Dependabot** | Bot GitHub care automatizeaza update-uri de dependinte |

---

## 14. Istoric modificări

| Data | Component afectat | Modificare |
|---|---|---|
| 2026-08-09 | Server TEST (test.peviitor.ro) | **Upgrade hardware:** RPi 4 (2 GB) → RPi 4 (8 GB). Cardul microSD păstrat. RAM: 1.8 GB usable → 8 GB (7.6 GiB usable). Adăugat swap 4 GB. IP local: 192.168.1.141 → 192.168.1.142. Kernel: 6.12 → 6.18.39+rpt-rpi-v8. |

---

*Document generat pe baza configuratiei hardware live, a documentului SAD si a inventarelor din repository.*
