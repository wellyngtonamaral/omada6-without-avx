# omada6-without-avx

## Introduction

Starting with MongoDB 5.0, official builds require CPUs with the **AVX instruction set**. This requirement continues in MongoDB 6.x and later. Many older servers and entry-level CPUs lack AVX, making it impossible to run the default Omada Controller v6 image with its internal MongoDB.

This project provides a solution: **replace MongoDB with FerretDB**, a MongoDB-compatible proxy that stores data in PostgreSQL. This allows Omada Controller v6 to run without AVX while maintaining compatibility with MongoDB drivers.

---

## AVX Requirement (How to test your CPU)

To check if your CPU supports AVX, run:

```bash
lscpu | grep -i avx
```

If nothing is returned, your CPU does **not** support AVX.

---

## Alternatives

If your CPU does not support AVX, you have two main options:

1. **Use an older MongoDB version (e.g., 4.4)** or a patched image without AVX (not recommended for production).
2. **Use FerretDB + PostgreSQL** (recommended), which provides MongoDB protocol compatibility without AVX requirements.

---

## Project Overview

This project runs Omada Controller v6 with FerretDB and PostgreSQL using Docker Compose. Omada is configured to use an external MongoDB URI, which points to FerretDB.

Services included:
- **postgres**: PostgreSQL with DocumentDB extension.
- **ferretdb**: MongoDB-compatible proxy.
- **controller**: Omada Controller v6 configured for external DB.

---

## Quick Start

### 1) Create `.env` file:

```dotenv
POSTGRES_USER=omada
POSTGRES_PASSWORD=super_secret
POSTGRES_DB=postgres # Do not change this
MONGODB_HOST=mondodb_hostname
OMADA_DB=omada
```

### 2) Use the provided `docker-compose.yml`:

```yaml
services:

  postgres:
    image: ghcr.io/ferretdb/postgres-documentdb:17
    restart: unless-stopped
    user: "1001:110" # Use your host user's UID and GID
    environment:
      - PUID=1001
      - PGID=110
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - ./db:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -h 127.0.0.1"]
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 5s

  ferretdb:
    image: ghcr.io/ferretdb/ferretdb:2.7.0
    restart: unless-stopped
    ports:
      - 27017:27017
    environment:
      - PUID=1001
      - PGID=110
      - FERRETDB_POSTGRESQL_URL=postgres://${POSTGRES_USER}:${POSTGRES_PASSWORD}@postgres:5432/${POSTGRES_DB}
    depends_on:
      postgres:
        condition: service_healthy

  controller:
    image: mbentley/omada-controller:6.0
    restart: unless-stopped
    ulimits:
      nofile:
        soft: 4096
        hard: 8192
    stop_grace_period: 60s
    network_mode: host
    environment:
      - PUID=1001
      - PGID=110
      - MANAGE_HTTP_PORT=8088
      - MANAGE_HTTPS_PORT=8043
      - PORTAL_HTTP_PORT=8088
      - PORTAL_HTTPS_PORT=8843
      - PORT_APP_DISCOVERY=27001
      - PORT_ADOPT_V1=29812
      - PORT_UPGRADE_V1=29813
      - PORT_MANAGER_V1=29811
      - PORT_MANAGER_V2=29814
      - PORT_DISCOVERY=29810
      - PORT_TRANSFER_V2=29815
      - PORT_RTTY=29816
      - SHOW_SERVER_LOGS=true
      - SHOW_MONGODB_LOGS=false
      - TZ=America/Sao_Paulo
      - MONGO_EXTERNAL=true
      - EAP_MONGOD_URI=mongodb://${POSTGRES_USER}:${POSTGRES_PASSWORD}@{MONGODB_HOST}/omada
    volumes:
      - ./data:/opt/tplink/EAPController/data
      - ./logs:/opt/tplink/EAPController/logs
    depends_on:
      ferretdb:
        condition: service_healthy

networks:
  default:
    name: omada6
```

### 3) Launch the stack:

```bash
docker compose up -d
```

Access Omada at: `https://<host-ip>:8043`

---

## Configuration Details

- Omada uses `MONGO_EXTERNAL=true` and `EAP_MONGOD_URI` to connect to FerretDB.
- Adjust networking and volumes as needed.
- Ensure ports for Omada services and FerretDB (27017) are reachable.

---

## Migration & Data Strategy

The initial strategy is to perform a **fresh installation** of Omada Controller v6 using this setup. Migration approaches should be evaluated **case by case**, depending on your environment and data requirements.

**Important:** Always create a **backup of your source controller** before attempting any migration.

In my tests, using the **built-in migration feature of Omada Controller** worked without major issues. However, results may vary, so plan and validate carefully.

For official migration details and upgrade notes, refer to the maintainer's repository:

[Omada Controller Migration Guide](https://github.com/mbentley/docker-omada-controller)

---
---
