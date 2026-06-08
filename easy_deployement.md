# EEW_Sensor Deployment Guide

This project is configured with a fully automated, "plug-and-play" Docker deployment pipeline. You do not need to clone the repository or manually build the code on your production server.

## 1. How It Works

1. **Automatic Builds:** Whenever code is pushed or merged into the `main` branch, GitHub Actions automatically builds the backend and frontend into a Docker image.
2. **Release Creation:** GitHub automatically tags the build with the date and run number, publishes it to the GitHub Container Registry (`ghcr.io`), and creates a GitHub Release.
3. **Runtime Variables:** The Docker image is completely public. Your actual secrets are injected from the `.env` file when the container boots up on your server.

---

## 2. Server Setup (First Time Only)

To deploy the application to a new server, you only need to create a single directory with two files: `.env` and `docker-compose.yml`.

### Step A: Authenticate Docker

Log in to GitHub Container Registry on your server using a Personal Access Token (PAT) with `read:packages` permissions:

```bash
docker login ghcr.io -u <your-github-username>
```

### Step B: Generate JWK Keys

The backend requires a pair of JSON Web Keys (JWKs) to securely sign and verify JWT tokens. You can easily generate these directly on your server using the official Deno Docker image by running the following command:

```bash
docker run --rm denoland/deno:alpine run https://raw.githubusercontent.com/Bimsara-Janakantha/EEW_Sensor/main/scripts/generate-jwk.js
```

This will output your `PRIVATE_JWK` and `PUBLIC_JWK`. Keep these handy for the next step.

### Step C: Create the Directory and Env File

```bash
mkdir -p /root/CRISiSLab/Deployment
cd /root/CRISiSLab/Deployment
nano backend.env
```

Add your production secrets to `backend.env` (replace the JWKs with the ones you just generated):

```env
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=your_secure_db_password
DATABASE_NAME=sensor_data
DATABASE_HOST=timescaledb
UDP_PORT=2098
HTTP_PORT=8080
DEV=0
SHOULD_STORE=1
PRIVATE_JWK={"use":"sig", "kty": "EC",  "kid": "...",  "crv": "P-256",  "x": "...",  "y": "...",  "d": "..."}
PUBLIC_JWK={"use":"sig", "kty": "EC",  "kid": "...",  "crv": "P-256",  "x": "...",  "y": "..."}
CREATE_DEFAULT_USER=1
DEFAULT_USER_PASSWORD=your_one_time_bootstrap_password
READ_FS_SIZE=0
```

### Step D: Create Docker Compose

If you are using the **Unified Deployment Approach** for multiple services, add the `timescaledb` and `ingest-deno` blocks to your master `docker-compose.yml`:

```bash
nano docker-compose.yml
```

```yaml
services:
  timescaledb:
    image: timescale/timescaledb:latest-pg14
    container_name: timescaledb
    ports:
      - "5432:5432"
    volumes:
      - timescale_data:/var/lib/postgresql/data
    environment:
      POSTGRES_USER: ${DATABASE_USERNAME}
      POSTGRES_PASSWORD: ${DATABASE_PASSWORD}
      POSTGRES_DB: ${DATABASE_NAME}

  ingest-deno:
    image: ghcr.io/bimsara-janakantha/eew_sensor:latest
    container_name: ingest-server
    env_file:
      - backend.env
    ports:
      - "8080:8080"
      - "2098:2098/udp"
    depends_on:
      - timescaledb
    entrypoint:
      [
        "/bin/sh",
        "-c",
        "chmod +x ./wait-for-db.sh && ./wait-for-db.sh timescaledb:5432 -- deno run --allow-ffi --allow-net --allow-read --allow-env --allow-run --allow-sys --unstable-cron --unstable-net server.ts",
      ]

volumes:
  timescale_data:

networks:
  default:
    name: CRISiSLab_EEW
```

> **Important**: Notice that `backend.env` is used for `ingest-deno`, but `timescaledb` directly uses variables like `${DATABASE_USERNAME}`. To make those variables available to `docker-compose` itself, you should put those specific DB variables in a `.env` file right next to `docker-compose.yml` so docker compose can read them, or replace them directly in the YAML file.

### Step E: Start the Server

Pull the latest images and run the containers in the background:

```bash
docker compose pull
docker compose up -d
```

---

## 3. Automated Daily Updates

To ensure your server always runs the latest release pushed to the `main` branch, set up a daily auto-update script.

### Create `auto_update.sh`

```bash
nano auto_update.sh
chmod +x auto_update.sh
```

Add the following content:

```bash
#!/bin/bash
DEPLOY_DIR="/root/CRISiSLab/Deployment"
cd "$DEPLOY_DIR" || exit

echo "Pulling latest images..."
docker compose pull

echo "Applying updates..."
docker compose up -d

echo "Cleaning up old images..."
docker image prune -f
```

### Schedule the Cron Job

Open the crontab editor:

```bash
crontab -e
```

Add this line to run the update every day at 00:00 AM:

```bash
00 00 * * * /root/CRISiSLab/Deployment/auto_update.sh >> /root/CRISiSLab/Deployment/eew_auto_update.log 2>&1
```

> **Note:** Docker Compose is smart enough to only restart containers if their underlying GitHub image has actually changed. If no new release was made, there will be zero downtime.
