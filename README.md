<div align="center">

![Traefik Architecture](https://img.shields.io/badge/Traefik-Architecture-blue?style=for-the-badge&logo=traefik)
![GitHub Pages](https://img.shields.io/badge/Deployed%20on-GitHub%20Pages-success?style=for-the-badge&logo=github)

**Interactive visualization of a Zero Trust Architecture using Traefik**

[🚀 **Live Demo**](https://youkyi.github.io/traefik-archi/) | [📖 **Documentation**](#features)

</div>

---

# Traefik Configuration - [YouKyi's] Infrastructure

This repository contains the base configuration and templates for [YouKyi's]'s Traefik instances. It serves as the reference point for deploying *Reverse Proxies* within a secure architecture.

## 🌳 Git Strategy & Branches

**Warning:** This repository operates on a **branch-based configuration model**.

* **`main` Branch (or `base`)**: Contains common configuration, global middlewares, and standard static configuration. **Do not deploy directly to production.**
* **Machine Branches (e.g., `srv-waf-01`, `srv-worker-01`)**: Each Traefik instance has its own branch. This is where routers specific to the services hosted on that machine are defined.

### Recommended Workflow
1.  Pull changes from the base: `git merge origin/main`
2.  Apply local specifics in `dynamic/routers.yml` or `dynamic/services.yml`.
3.  Deploy to the target machine (git pull + container restart).

---

## 🏛️ Architecture & Roles

The infrastructure relies on strict traffic segmentation. **Traefik Central** is **NEVER** directly exposed to the Internet.

### 1. Traefik Central (Internal Proxy)
* **Role:** Single entry point for the *internal* infrastructure.
* **Exposures:** LAN / VPN only. No WAN exposure.
* **Functions:**
    * Manages Wildcard certificates (`*.int.example.com`).
    * Centralizes authentication (Authentik).
    * Routes traffic to *Local Traefiks* via HTTPS.
    * Exposes monitoring APIs of local Traefiks via `proxy-services.yml` and `proxy-routers.yml`.

### 2. Traefik Local (Sidecar / Node)
* **Role:** Local TLS termination on each machine/VM hosting services.
* **Functions:**
    * Receives encrypted traffic.
    * Forwards to the final service (usually via HTTP on localhost/docker network).
* **Security (AllowList):**
    * Only accepts connections from **Traefik Central** (for internal services).
    * OR only accepts connections from the **WAF** (for public services).

### 3. BunkerWeb (Edge WAF)
* **Role:** Perimeter security (Edge).
* **Exposure:** Public (WAN).
* **Function:** Filters attacks and forwards *clean* traffic to the relevant **Local Traefiks**. It **never** communicates with Traefik Central.

---

## 📂 File Structure

The directory structure separates static configuration from dynamic configuration.

```text
.
├── traefik.yml                  # Static Configuration (EntryPoints, Logs, Providers)
└── traefik/
    └── dynamic/                 # Dynamic Configuration (Hot Reload)
        ├── middlewares.yml      # GENERAL Middlewares (Auth, Headers, Compress)
        ├── middlewares-back.yml # LOCAL Middlewares (IP AllowLists, Retry)
        ├── routers.yml          # Standard service routers
        ├── services.yml         # Standard backends
        ├── wildcard.yml         # Template for Wildcard certificate
        │
        # Files specific to TRAEFIK CENTRAL only: # not visible in this repository
        ├── proxy-routers.yml    # Routers to access remote Traefik APIs
        └── proxy-services.yml   # Services pointing to remote Traefik APIs

```

---

## 🛡️ Traffic Management & Middlewares

It is imperative to choose the middleware chain corresponding to the traffic source.

### 1. For Traefik Central (Internal User Traffic)

Use chains defined in `middlewares.yml`.

* **`chain-standard`**: Security Headers + Compression (Internal Public).
* **`chain-standard-authentik`**: Headers + Compression + **Authentication**. (Standard for admin tools).

### 2. For Traefik Local (Infrastructure Traffic)

Use chains defined in `middlewares-back.yml`. These proxies validate the request origin.

* **`chain-back-standard`**:
* Usage: Internal services.
* Restriction: Authorizes ONLY the **Traefik Central** IP (`sec-allow-front-proxy-only`).


* **`chain-back-waf`**:
* Usage: Web-exposed services.
* Restriction: Authorizes ONLY the **Edge WAF** IPs (`sec-allow-front-waf-only`).


* **`chain-back-auth`**:
* Usage: Critical internal services.
* Restriction: Central IP + Double Authentik validation.



### 3. Monitoring & Metrics

Specific middlewares are designed to protect `/metrics` endpoints (Prometheus).

* **`chain-prometheus-restricted`**:
* Usage: On routers exposing metrics.
* Restriction: Authorizes ONLY the Prometheus server IP (`sec-allow-prometheus-only`) + Standard headers.



---

## 📝 Adding a New Service

### Case 1: Internal Service (via Traefik Central)

1. **On Traefik Central** (File `proxy-routers.yml` or `routers.yml`): Create a route pointing to the Local Traefik.
2. **On Traefik Local** (Machine-specific branch):
* Edit `routers.yml` (or Docker labels).
* Apply the restriction middleware:



```yaml
# Example: Internal service on a local machine
http:
  routers:
    internal-app:
      rule: Host(`app.int.example.com`)
      service: my-app
      entryPoints:
        - websecure
      middlewares:
        - chain-back-standard # <-- REJECTS anything not coming from Central

```

### Case 2: Public Service (via BunkerWeb)

1. Configure the WAF to point to the local machine's IP.
2. **On Traefik Local**:
* Apply the WAF middleware:



```yaml
# Example: Public service
http:
  routers:
    public-app:
      rule: Host(`www.public-domain.com`)
      service: my-site
      middlewares:
        - chain-back-waf # <-- REJECTS anything not coming from WAFs

```

### Case 3: Docker labels

1. add this label to your docker-compose.yml file. 

```yaml
    labels:
      - traefik.enable=true
      - traefik.http.routers.app.service=app
      - traefik.http.routers.app.rule=Host(`domaine.com`)
      - traefik.http.routers.app.entrypoints=websecure
      - traefik.http.routers.app.tls=true
      - traefik.http.routers.app.middlewares=middleware-name@file
```
