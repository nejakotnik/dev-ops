# Avtomatizirano razvojno in testno okolje za Report App

Ta repozitorij vsebuje **tri različne avtomatizirane rešitve** za postavitev aplikacije **Report App**:

1. **Vagrant + Ansible + VirtualBox** – lokalno razvojno okolje
2. **Cloud-init + Multipass** – hitra postavitev VM-ja
3. **Docker + Ansible (Infra as Code)** – kontejnerska različica z Docker Compose in GHCR

Cilj projekta je omogočiti **ponovljivo, enostavno in zanesljivo** postavitev celotnega okolja (backend, frontend, baza, reverse proxy, varnost) z enim samim ukazom.

> ⚠️ **Vsa gesla in certifikati v projektu so namenjeni izključno razvoju in testiranju.**

---

## Namen projekta

Namen projekta je prikazati **celovito avtomatizacijo postavitve sodobnega aplikacijskega stacka** z uporabo orodij in praks, ki se uporabljajo v realnih DevOps okoljih. Projekt združuje virtualizacijo, konfiguracijski management, kontejnerizacijo, CI/CD in varnost ter omogoča hitro, ponovljivo in zanesljivo postavitev razvojnega ali demo okolja.

Projekt je razvit kot del študijskih vaj in služi kot:

- demonstracija znanja iz **virtualizacije in avtomatizacije**,
- osnova za **lokalni razvoj ali predstavitev aplikacije**,
- primer dobre prakse za **Infrastructure as Code (IaC)**.

---

## Tehnološki pregled

Projekt uporablja naslednje tehnologije:

- **Ubuntu 22.04 (Jammy)**
- **FastAPI (Python backend)**
- **Angular (frontend)**
- **PostgreSQL**
- **Redis**
- **Nginx** (reverse proxy)
- **Docker + Docker Compose** (za kontejnersko različico)
- **TLS certifikati (lokalni/self-signed ali Let's Encrypt)**
- **systemd** za avtomatski zagon backend servisa
- **UFW požarni zid**
- **XFCE + XRDP** (grafični dostop preko RDP)
- **CI/CD** z GitHub Actions za avtomatsko buildanje image-ov
- **Buildx / multi-stage Docker build** za optimizacijo image-ov

---

# Možnost A: Vagrant + Ansible + VirtualBox

Ta možnost je namenjena predvsem **lokalnemu razvoju** in omogoča popolnoma avtomatizirano postavitev z ukazom:

```bash
vagrant up
```

## Zahteve

- VirtualBox
- Vagrant

## Zagon
```bash
cd dev-ops
vagrant up
```
Ob prvem zagonu se:
- ustvari VM
- namesti Ansible
- izvede Ansible playbook
- zažene celotna aplikacija

### Struktura projekta
```
dev-ops/
├── Vagrantfile
├── provision-base.sh
└── ansible/
    ├── inventory
    └── playbook.yml
```

### Kaj se samodejno nastavi

#### Sistem

- Ubuntu 22.04
- Časovni pas
- UFW požarni zid

#### Storitve

- Nginx (reverse proxy + static)
- PostgreSQL (baza + uporabnik)
- Redis
- systemd servis za FastAPI

#### Backend

- Python virtualno okolje
- FastAPI aplikacija
- `/etc/reportapp.env` konfiguracija

#### Frontend

- Node.js (NodeSource)
- `npm install`
- Angular production build

#### TLS

- Lokalni self-signed CA in certifikat

#### Grafični dostop

- XFCE
- XRDP

### Dostop

#### Aplikacija

- HTTP: http://localhost:8080
- HTTPS: https://localhost:8443

#### API

- https://localhost:8443/api/

#### RDP
- localhost:33389

# Možnost B: Cloud-init + Multipass

Omogoča **hitro postavitev VM-ja** z uporabo `cloud-init` brez Ansible-a.

### Struktura

```
cloud-init/
└── cloud-init.yml
```

### Kaj naredi cloud-init

- Namesti vse sistemske pakete
- Ustvari uporabnika `reportapp`
- Nastavi PostgreSQL bazo
- Klonira Report App repozitorij
- Pripravi Python virtualenv
- Zgradi Angular frontend
- Ustvari TLS certifikate
- Konfigurira Nginx
- Nastavi UFW
- Ustvari systemd servis
- Po želji namesti XFCE
- Po zagonu VM-ja zažene aplikacijo

### Zagon z Multipass

```bash
multipass launch \
  --name reportapp \
  --disk 30G \
  --memory 8G \
  --cpus 4 \
  --cloud-init "cloud-init/cloud-init.yml" \
  --timeout 1800
```

### Dostop

#### SSH

```bash
multipass shell reportapp
multipass info reportapp
```

### Aplikacija

- http://<VM-IP>/
- http://<VM-IP>/api/

# Možnost C: Docker + Ansible (Infra as Code)

Ta možnost uvaja **kontejnersko arhitekturo** za Report App in je namenjena bolj produkcijsko podobnemu okolju. Celoten stack (frontend, backend, scheduler, PostgreSQL, Nginx) teče v Docker kontejnerjih, orkestriranih z **Docker Compose**.

Namestitev in zagon sta avtomatizirana z Ansible playbookom.

### Struktura (Docker / Infra)

```
vagrant/
├── ansible/
│ └── playbook-docker.yml
└── infra/
├── docker-compose.yml
└── conf.d/
├── default.conf
└── timeout.conf
```

### playbook-docker.yml

Ansible playbook, ki:

- namesti Docker Engine, Docker Compose plugin in Buildx
- klonira `dev-ops` repozitorij na VM (`/opt/reportapp`)
- preveri prisotnost `docker-compose.yml` in Nginx konfiguracije
- pripravi direktorije za:
  - audit loge
  - generirana poročila
  - TLS certifikate
- generira **self-signed TLS certifikat** (če ne obstaja)
- prijavi VM v **GitHub Container Registry (GHCR)**
- izvede `docker compose pull` in `docker compose up -d`

Playbook je idempotenten in primeren za večkratni zagon.

### docker-compose.yml

Definira celoten aplikacijski stack:

- **postgres** – PostgreSQL 16 (persistent volume)
- **backend** – FastAPI aplikacija (GHCR image)
- **scheduler** – ločen worker za periodične naloge
- **frontend** – Angular frontend (GHCR image)
- **nginx** – reverse proxy + TLS terminacija

Posebnosti:

- uporaba `.env` datoteke za občutljive nastavitve
- ločeni kontejnerji za backend in scheduler
- skupno Docker omrežje `avichron-net`

### Nginx konfiguracija (Docker)

#### default.conf

- HTTP → HTTPS preusmeritev
- `/api/` → FastAPI backend (`backend:8000`)
- `/` → Angular frontend
- SPA fallback za Angular deep-linke
- podpora za večje zahteve (`client_max_body_size 50m`)

#### timeout.conf

Poveča proxy timeout vrednosti (primerno za dolgotrajne reporte):

- `proxy_connect_timeout 600`
- `proxy_read_timeout 600`
- `proxy_send_timeout 600`

### TLS

- Docker okolje uporablja **self-signed certifikat**, generiran z Ansible
- Certifikat je mountan v Nginx kontejner
- Namenjeno izključno razvoju / testiranju

### Zagon (Docker varianta)

Po zagonu VM-ja:

```bash
ansible-playbook vagrant/ansible/playbook-docker.yml
```

Aplikacija je nato dostopna na:

- https://localhost/
- https://localhost/api/

## CI/CD pipeline (GitHub Actions)

Docker image-i za backend in frontend se **samodejno gradijo in objavljajo** v **GitHub Container Registry (GHCR)** z uporabo **GitHub Actions**.

Pipeline vključuje:

- multi-stage Docker build (optimizirana velikost image-a)
- uporabo **Docker Buildx**
- taganje image-ov (`latest`, commit SHA)
- objavo v GHCR (`ghcr.io/eghz23/...`)

Ob vsaki spremembi v aplikacijskem repozitoriju se image samodejno posodobi. Deploy ni avtomatiziran – VM image-e pridobi z `docker compose pull`.

## Dokumentacija in dokazila

Za izpolnitev zahtev naloge je priporočeno (in delno obvezno):

- 📸 **Screenshoti**:
  - `vagrant up` (uspešen zagon)
  - cloud-init provisioning (Multipass)
  - delujoča aplikacija v brskalniku (HTTP/HTTPS)
  - RDP / XFCE dostop (če uporabljen)

- 🎥 **Kratek video (neobvezno)**:
  - prikaz avtomatskega zagona okolja

- 🔗 **Povezave**:
  - Backend Dockerfile (multi-stage build)
  - Frontend Dockerfile
  - GitHub Actions workflow

## Javni dostop (deployment)

Aplikacijski stack je mogoče brez sprememb deployati na:

- fakultetne `devops-sk-XX` VM-je
- osebne VM-je
- javne cloud ponudnike (npr. Oracle Free Tier)

Za javni dostop je potrebno:

- odpreti porte 80/443
- nastaviti DNS (neobvezno)

## Opombe

- Projekt je namenjen **razvoju in testiranju**, ne produkciji
- TLS certifikati so **lokalni in nezaupljivi**
- Gesla so trdo kodirana zgolj za lažjo uporabo
- Obe možnosti postavitve vzpostavita **funkcionalno enako okolje**
- Docker varianta je najbolj podobna produkciji
- Image-i se pridobijo iz **GitHub Container Registry (GHCR)**
- `.env` datoteka mora obstajati (lahko prazna)



