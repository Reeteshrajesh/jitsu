## 📦 Jitsu Production Deployment on EC2 (Single Instance)

This guide helps you deploy the full [Jitsu](https://jitsu.com) open-source stack (Ingest, Console, Rotor, Bulker, Kafka, ClickHouse, Postgres, MongoDB, NGINX) on a new **Amazon EC2 instance** using **Docker Compose**.

---

### 🗂️ Folder Structure

```
jitsu-prod/
├── .env                     # Sensitive credentials (DO NOT commit to git)
├── docker-compose.yml       # All services defined here
├── nginx.conf               # NGINX reverse proxy + SSL termination
└── ssl/                     # Place SSL certs here (fullchain.pem, privkey.pem)
```

---

## 🚀 EC2 Instance Setup

### 1. Launch EC2 Instance

* **Region**: Choose your preferred region
* **AMI**: Ubuntu 22.04 LTS
* **Instance Type**: `t3.large` or higher recommended
* **Key Pair**: Create or select existing
* **Storage**: At least 30 GB GP3
* **Security Group**: Configure as follows:

| Port | Protocol | Source    | Purpose       |
| ---- | -------- | --------- | ------------- |
| 22   | TCP      | Your IP   | SSH           |
| 80   | TCP      | 0.0.0.0/0 | HTTP (NGINX)  |
| 443  | TCP      | 0.0.0.0/0 | HTTPS (NGINX) |

---

## 🔐 Setup on EC2 (Ubuntu)

### SSH into your EC2:

```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

### 1. Install Docker & Docker Compose

```bash
sudo apt update && sudo apt install -y docker.io docker-compose
sudo usermod -aG docker $USER
newgrp docker
```

### 2. Clone Project and Setup Folder

```bash
git clone https://github.com/your-org/jitsu-prod.git
cd jitsu-prod
```

> Or manually create structure:

```bash
mkdir jitsu-prod && cd jitsu-prod
touch .env docker-compose.yml nginx.conf
mkdir ssl
```

### 3. Add SSL Certificates

Use **Certbot** or manually generate:

```bash
sudo apt install certbot
sudo certbot certonly --standalone -d your-domain.com
```

Then copy the cert files:

```bash
cp /etc/letsencrypt/live/your-domain.com/fullchain.pem ./ssl/
cp /etc/letsencrypt/live/your-domain.com/privkey.pem ./ssl/
```

---

## ⚙️ Configuration

### 1. Fill `.env`

```bash
nano .env
```

```env
# Example values (change securely!)
POSTGRES_USER=jitsu
POSTGRES_PASSWORD=supersecure
POSTGRES_DB=jitsu
MONGO_USER=admin
MONGO_PASSWORD=securemongo
CLICKHOUSE_PASSWORD=clickhouse123
SERVICE_AUTH_TOKEN=service-admin-account:secret
RAW_AUTH_TOKEN=yourtoken
DOMAIN_NAME=your-domain.com
```

### 2. Add Config Files

Copy-paste:

* [`docker-compose.yml`](#)
* [`nginx.conf`](#)
  into the project directory.

---

## ▶️ Start Services

```bash
docker-compose pull
docker-compose up -d
```

---

## ✅ Verify Deployment

| URL                                | Description        |
| ---------------------------------- | ------------------ |
| `https://your-domain.com/`         | Ingest endpoint    |
| `https://your-domain.com/_console` | Jitsu Console (UI) |

---

## 🛡️ Security Checklist

* ✅ Use `.env` and never commit it to Git
* ✅ Open only necessary ports
* ✅ Set complex auth tokens and DB passwords
* ✅ Use HTTPS with real certificates
* ✅ Disable root login (optional hardening)
* ✅ Run `ufw enable` to enable basic firewall (optional)

---

## 🧼 Maintenance & Tips

* SSL auto-renew:

  ```bash
  sudo crontab -e
  0 0 * * * certbot renew --quiet && docker restart nginx
  ```

* View logs: `docker-compose logs -f`

* Restart all services: `docker-compose restart`

* Stop stack: `docker-compose down`

---

Let me know if you want:

* Systemd service to auto-start on reboot
* Backup scripts for Postgres, ClickHouse, Mongo
* Automated deployment with Ansible or Terraform
