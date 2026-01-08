# n8n Production Setup Guide (GCP Free Tier + Supabase + ngrok)

## Architecture

Internet -\> https://`<ngrok-domain>`{=html} -\> ngrok tunnel -\>
localhost:5678 -\> Docker n8n -\> Supabase Postgres

------------------------------------------------------------------------

## VM Preparation (GCP e2-micro)

### Enable Swap (2GB)

``` bash
sudo fallocate -l 2G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

Verify:

``` bash
free -h
```

------------------------------------------------------------------------

## Install Docker

``` bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER
newgrp docker
```

Test:

``` bash
docker run hello-world
```

------------------------------------------------------------------------

## Supabase Setup

Use **Connection Pooler** only.

  Field   Value
  ------- ---------------------------------------------
  Host    aws-1-`<region>`{=html}.pooler.supabase.com
  Port    6543
  User    postgres.`<project_ref>`{=html}
  DB      postgres

Network restrictions: add VM IP as `x.x.x.x/32`

Pooler settings:

-   Pool size: 30
-   Pool mode: transaction

Run in SQL editor:

``` sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
ALTER DATABASE postgres SET timezone='UTC';
CREATE SCHEMA IF NOT EXISTS n8n AUTHORIZATION postgres;
GRANT ALL PRIVILEGES ON SCHEMA n8n TO postgres;
```

------------------------------------------------------------------------

## docker-compose.yml

``` yaml
version: '3.8'
services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=aws-1-region.pooler.supabase.com
      - DB_POSTGRESDB_PORT=6543
      - DB_POSTGRESDB_DATABASE=postgres
      - DB_POSTGRESDB_SCHEMA=n8n
      - DB_POSTGRESDB_USER=postgres.projectref
      - DB_POSTGRESDB_PASSWORD=YOUR_DB_PASSWORD
      - DB_POSTGRESDB_SSL_REJECT_UNAUTHORIZED=false
      - DB_POSTGRESDB_MAX_CONNECTIONS=10
      - N8N_ENCRYPTION_KEY=GENERATED_KEY
      - N8N_HOST=your-ngrok-domain.ngrok-free.dev
      - N8N_PORT=443
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://your-ngrok-domain.ngrok-free.dev
      - GENERIC_TIMEZONE=Asia/Kolkata
      - N8N_SECURE_COOKIE=true
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

Generate encryption key:

``` bash
openssl rand -hex 32
```

Start n8n:

``` bash
docker compose up -d
```

------------------------------------------------------------------------

## ngrok Setup

Install:

``` bash
curl -s https://ngrok-agent.s3.amazonaws.com/ngrok.asc | sudo tee /etc/apt/trusted.gpg.d/ngrok.asc >/dev/null
echo "deb https://ngrok-agent.s3.amazonaws.com buster main" | sudo tee /etc/apt/sources.list.d/ngrok.list
sudo apt update
sudo apt install ngrok -y
```

Add token:

``` bash
ngrok config add-authtoken YOUR_TOKEN
```

Run tunnel:

``` bash
ngrok http 5678
```

------------------------------------------------------------------------

## Run ngrok in background

Create service:

``` bash
sudo nano /etc/systemd/system/ngrok.service
```

``` ini
[Unit]
Description=ngrok tunnel for n8n
After=network.target docker.service

[Service]
User=YOUR_USERNAME
ExecStart=/usr/bin/ngrok http 5678
Restart=always

[Install]
WantedBy=multi-user.target
```

Enable:

``` bash
sudo systemctl daemon-reload
sudo systemctl enable ngrok
sudo systemctl start ngrok
```

------------------------------------------------------------------------

## Verify

Open:

    https://your-ngrok-domain.ngrok-free.dev

Check logs:

``` bash
docker logs n8n
```

------------------------------------------------------------------------

## Key Learnings

-   Always use Supabase pooler (not direct DB).
-   Swap is mandatory for 1GB VM.
-   `N8N_PORT=443` is for public URL generation, not container binding.
-   ngrok forwards to local 5678.
-   Encryption key must never change.
-   All data lives in Supabase, not Docker.
