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

### 2) Use the provided `docker-compose.yml` and `.env`:

Clone this repository and navigate to its directory:

```bash
git clone https://github.com/wellyngtonamaral/omada6-without-avx.git
cd omada6-without-avx
```

Ensure you have created the `.env` file as shown in step 1. The `docker-compose.yml` is already included in the repository, so you can start the stack directly.

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

## References

- [Omada Controller GitHub](https://github.com/mbentley/docker-omada-controller)
- [MongoDB Official Site](https://www.mongodb.com)
- [PostgreSQL Official Site](https://www.postgresql.org)
- [FerretDB Official Site](https://www.ferretdb.io)
