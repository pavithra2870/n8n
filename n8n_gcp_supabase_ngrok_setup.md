# n8n Production Setup Guide (GCP Free Tier + Supabase + ngrok)

## Architecture

Internet -\> https://`<ngrok-domain>`{=html} -\> ngrok tunnel -\>
localhost:5678 -\> Docker n8n -\> Supabase Postgres

------------------------------------------------------------------------

## How to make use of Always Free tier e2.micro VM Instance

To ensure your Google Compute Engine instance remains part of the **Free Tier**, you must adhere strictly to specific hardware and regional requirements.

### 1. Select a Valid Region

The Free Tier is only available in the following US regions. Selecting any other region will result in hourly charges.

* **Oregon:** `us-west1`
* **Iowa:** `us-central1`
* **South Carolina:** `us-east1`

### 2. Machine Configuration

* **Series:** E2
* **Machine Type:** `e2.micro` (2 vCPU, 1 GB RAM)

### 3. Boot Disk Settings

Your free storage limit is **30 GB** total per month across all your instances.

* **Size:** 30 GB or less.
* **Type:** Standard Persistent Disk (`pd-standard`).
* *Warning:* Do **not** select SSD or Balanced Persistent Disks, as these are not covered under the free tier.

* **Operating System:** Public images like Debian, CentOS, or Ubuntu are free. Avoid "Premium" images (like Red Hat or Windows Server) which carry licensing costs.

### 4. Networking and External IP

* **External IP:** Ensure you use a **Standard Tier** network service tier if you want to keep costs at zero.
* **Usage:** One free ephemeral external IP address is included per month.
* **Egress:** You get **1 GB** of free outbound data transfer per month (excluding China and Australia).
---

### 5. Firewall

In the Firewall section of the VM creation page, check the boxes for:

Allow HTTP traffic

Allow HTTPS traffic

To create custom rules (e.g., for a specific port like 8080), navigate to VPC Network > Firewall and create a rule with:

Targets: All instances in the network.

Source IP ranges: 0.0.0.0/0 (Allows everyone).

Protocols and ports: Check tcp and enter the port number.

### 6. Snapshots

Go to Compute Engine > Storage > Snapshots.

Click Create Snapshot.

Source Disk: Select your e2-micro boot disk.

Location: Choose Regional (and pick the same region as your VM) to keep costs lower than "Multi-regional."

Retention days is preset to 14. You can keep it as 3 for lower costs if you want.

While no backup costs $0, it is recommended to have snapshots

### How to Verify it's Free

When configuring the VM in the Google Cloud Console, look at the **Estimated Cost** box on the right side of the screen. While it may show a monthly estimate (e.g., $7.00), there should be a note underneath stating:

> "Your Free Tier usage credit will be applied to this instance."

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
## To get your free domain from ngrok

- signup: https://ngrok.com/
- use developer role
- when you have onboarded you get your free domain on Domains in the sidebar
- click on authToken in the sidebar to get your authToken 
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

## 1. Install `screen`

```bash
sudo apt update
sudo apt install screen -y
```

---

## 2. Start a named screen session for ngrok

```bash
screen -S ngrok
```

You are now inside a persistent terminal.

---

## 3. Run ngrok inside the screen session

```bash
ngrok http 5678
```

You will see ngrok start and print the public forwarding URL.

---

## 4. Detach the screen (keep it running in background)

Press:

```
Ctrl + A  then  D
```

ngrok is now running safely in background.

---

## 5. Re-attach later anytime

```bash
screen -r ngrok
```

---

## 6. Check active screen sessions

```bash
screen -ls
```

---

## 7. Stop ngrok completely

```bash
screen -r ngrok
```

Then inside it:

```bash
Ctrl + C
exit
```

---

## 8. Auto-start ngrok on reboot (screen + crontab)

Edit root crontab:

```bash
sudo crontab -e
```

Add at bottom:

```bash
@reboot screen -dmS ngrok ngrok http 5678
```

Reboot once to test:

```bash
sudo reboot
```

After reboot:

```bash
screen -ls
```

You should see `ngrok` running.

---

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
-   Use supabase us region so that ur vm instance and supabase project are in same region (lesser latency)
-   NEVER LOSE YOUR N8N_ENCRYPTION_KEY - It is the key to your supabase database - you lose it, you lose all your credentials, workflows
