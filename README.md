# NeoTrack Local Server

![License](https://img.shields.io/badge/license-MIT-blue)
![Python](https://img.shields.io/badge/python-3.12-blue)
![Docker](https://img.shields.io/badge/docker-ready-blue)
![Traefik](https://img.shields.io/badge/Traefik-v3-green)

> Complete self-hosted replacement for the discontinued NeoTrack cloud
> infrastructure.

## Features

-   Complete NeoTrack API emulation
-   Docker & Portainer ready
-   Traefik reverse proxy
-   Local DNS interception
-   Automatic FIT extraction
-   Automatic FitTrackee import
-   Automatic cleanup
-   Ready for Strava synchronization

## High-level architecture

``` mermaid
flowchart LR
GPS[NeoTrack GPS]
DNS[Local DNS]
TR[Traefik]
API[Flask API]
DATA[/FIT Storage/]
FT[FitTrackee]
DB[(PostgreSQL)]

GPS --> DNS
DNS --> TR
TR --> API
API --> DATA
DATA --> FT
FT --> DB
```

## Repository

``` text
.
├── app.py
├── docker-compose.yml
├── requirements.txt
├── README.md
├── data/
│   └── *.fit
├── scripts/
│   ├── watcher.py
│   ├── importer.py
│   └── cleanup.py
├── fittrackee/
└── docs/
```

## Directory purpose

### app.py

Implements every NeoTrack endpoint.

-   GET /device/bonded/`<id>`{=html}
-   GET /device/dlist/`<id>`{=html}
-   POST /device/upload/`<id>`{=html}

Receives multipart uploads, extracts FIT files and stores them.

### data/

Temporary queue for uploaded FIT files.

### scripts/

Contains automation:

-   Watch filesystem
-   Import FIT into FitTrackee
-   Delete imported FIT

## Deployment

``` bash
git clone https://github.com/<your-account>/neotrack-server.git
cd neotrack-server
docker compose pull
docker compose up -d
```

## DNS interception

Configure your DNS server (AdGuard Home, Pi-hole or router).

Override:

``` text
upload.neostrack.com
corp.brytonsport.com
```

towards

``` text
192.168.1.151
```

Without this override the GPS contacts the discontinued cloud.

## AdGuard Home

DNS Rewrite

``` text
upload.neostrack.com -> 192.168.1.151
corp.brytonsport.com -> 192.168.1.151
```

## Pi-hole

Local DNS Records

``` text
upload.neostrack.com
corp.brytonsport.com
```

## Traefik

``` yaml
labels:
  - traefik.enable=true
  - traefik.docker.network=traefik_default
  - traefik.http.routers.neotrack.rule=Host(`upload.neotrack.com`)
  - traefik.http.routers.neotrack.entrypoints=web
  - traefik.http.services.neotrack.loadbalancer.server.port=5000
```

## Communication sequence

``` mermaid
sequenceDiagram
participant GPS
participant DNS
participant Traefik
participant API
participant FitTrackee

GPS->>DNS: Resolve upload.neostrack.com
DNS-->>GPS: 192.168.1.151
GPS->>Traefik: GET /device/bonded
Traefik->>API: Forward
API-->>GPS: 200 OK
GPS->>Traefik: GET /device/dlist
Traefik->>API: Forward
API-->>GPS: 200 OK
GPS->>Traefik: POST /device/upload
Traefik->>API: multipart/form-data
API->>API: Extract FIT
API->>FitTrackee: Import activity
FitTrackee-->>API: Success
API->>API: Delete FIT
API-->>GPS: 200 OK
```

## Troubleshooting

``` bash
docker logs -f neotrack-api
docker logs -f fittrackee
docker logs -f fittrackee-db
```

Verify DNS:

``` bash
nslookup upload.neostrack.com
```

## Roadmap

-   Strava sync
-   Garmin Connect
-   Komoot
-   Multi-user
-   Multi-device
-   Admin dashboard
-   Metrics
-   OTA support

## License

MIT
