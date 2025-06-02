* PostgreSQL + Redis backend,
* Jitsu Control Plane on port 7000,
* Jitsu Data Plane on port 8000,
* NGINX proxy with HTTPS for:

  * `/configurator` → Control Plane,
  * `/api` → Data Plane.

---

# Step 0: Prepare your server

Assuming a clean Ubuntu 20.04+ server with root or sudo access.

---

# Step 1: Install Docker & Docker Compose

Run:

```bash
# Update & install dependencies
sudo apt update && sudo apt upgrade -y
sudo apt install -y apt-transport-https ca-certificates curl software-properties-common

# Add Docker's official GPG key
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

# Add Docker repo
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] \
  https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine & Docker Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Add your user to docker group (logout/login required for effect)
sudo usermod -aG docker $USER

# Verify docker installed
docker --version
docker compose version
```

---

# Step 2: Create your project directory & files

```bash
mkdir -p ~/jitsu-setup
cd ~/jitsu-setup
```

Create `docker-compose.yml` with:

```yaml
version: '3.7'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: jitsu
      POSTGRES_PASSWORD: jitsu
      POSTGRES_DB: jitsu
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:6
    ports:
      - "6379:6379"

  jitsu:
    image: jitsucom/jitsu:latest
    depends_on:
      - redis
      - postgres
    environment:
      - REDIS_URL=redis://redis:6379
      - CONFIGURATOR_META_STORAGE=postgres
      - CONFIGURATOR_META_DSN=postgres://jitsu:jitsu@postgres:5432/jitsu?sslmode=disable
      - EVENTNATIVE_META_STORAGE=postgres
      - EVENTNATIVE_META_DSN=postgres://jitsu:jitsu@postgres:5432/jitsu?sslmode=disable
    ports:
      - "7000:7000"  # Control Plane
      - "8000:8000"  # Data Plane (event proxy)
      - "8001:8001"  # Debug UI (optional)

volumes:
  pgdata:
```

---

# Step 3: Start Docker Compose stack

```bash
docker compose up -d
```

Wait a minute for all containers to start.

Check containers status:

```bash
docker ps
```

You should see `postgres`, `redis`, and `jitsu` running.

---

# Step 4: Install NGINX

```bash
sudo apt install -y nginx
```

---

# Step 5: Set up NGINX reverse proxy config

Create a new NGINX site config file:

```bash
sudo nano /etc/nginx/sites-available/jitsu.conf
```

Paste this content (replace `activity.liquide.life` with your actual domain):

```nginx
server {
    listen 80;
    server_name activity.liquide.life;

    # Redirect HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name activity.liquide.life;

    ssl_certificate /etc/letsencrypt/live/activity.liquide.life/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/activity.liquide.life/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location /configurator/ {
        proxy_pass http://localhost:7000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://localhost:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Optional debug UI
    location /events/ {
        proxy_pass http://localhost:8001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

# Step 6: Enable NGINX config & reload

```bash
sudo ln -s /etc/nginx/sites-available/jitsu.conf /etc/nginx/sites-enabled/jitsu.conf

sudo nginx -t
sudo systemctl reload nginx
```

---

# Step 7: Obtain SSL certificates with Certbot

Install Certbot and the NGINX plugin:

```bash
sudo apt install -y certbot python3-certbot-nginx
```

Run Certbot to get and install SSL:

```bash
sudo certbot --nginx -d activity.liquide.life
```

Follow prompts, agree to redirect HTTP → HTTPS.

---

# Step 8: Final checks

* Visit in your browser:

  * `https://activity.liquide.life/configurator` — Jitsu Control Plane UI
  * `https://activity.liquide.life/api` — Should proxy to Data Plane event endpoint

* Test event proxy by integrating your client SDK with:

```js
jitsu('load', '<your_project_id>', {
  api_host: 'https://activity.liquide.life/api',
});
```

---

# Bonus: How to check logs & containers

```bash
docker logs -f jitsu_jitsu_1  # Assuming compose project name is jitsu
sudo journalctl -u nginx -f
```

---

# Summary files:

---

### docker-compose.yml

```yaml
version: '3.7'
services:
  postgres:
    image: postgres:13
    environment:
      POSTGRES_USER: jitsu
      POSTGRES_PASSWORD: jitsu
      POSTGRES_DB: jitsu
    volumes:
      - pgdata:/var/lib/postgresql/data
    ports:
      - "5432:5432"

  redis:
    image: redis:6
    ports:
      - "6379:6379"

  jitsu:
    image: jitsucom/jitsu:latest
    depends_on:
      - redis
      - postgres
    environment:
      - REDIS_URL=redis://redis:6379
      - CONFIGURATOR_META_STORAGE=postgres
      - CONFIGURATOR_META_DSN=postgres://jitsu:jitsu@postgres:5432/jitsu?sslmode=disable
      - EVENTNATIVE_META_STORAGE=postgres
      - EVENTNATIVE_META_DSN=postgres://jitsu:jitsu@postgres:5432/jitsu?sslmode=disable
    ports:
      - "7000:7000"
      - "8000:8000"
      - "8001:8001"

volumes:
  pgdata:
```

---

### NGINX config `/etc/nginx/sites-available/jitsu.conf`

```nginx
server {
    listen 80;
    server_name activity.liquide.life;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name activity.liquide.life;

    ssl_certificate /etc/letsencrypt/live/activity.liquide.life/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/activity.liquide.life/privkey.pem;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location /configurator/ {
        proxy_pass http://localhost:7000/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://localhost:8000/api/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /events/ {
        proxy_pass http://localhost:8001/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

---

# Let me know if you want help with:

* Automating setup with a shell script
* Troubleshooting errors or logs
* Configuring Jitsu SDK on your frontend
