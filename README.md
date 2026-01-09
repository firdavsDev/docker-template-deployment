# 🚀 Project Deployment Guide (AWS / Ubuntu Server)

## 1. Update & Prepare Server

```bash
sudo apt update && sudo apt upgrade -y
```

Install Git (if not already installed):

```bash
git --version || sudo apt install -y git
```

---

## 2. Install Docker

Remove old versions (if any):

```bash
sudo apt-get remove docker docker-engine docker.io containerd runc -y
```

Install prerequisites:

```bash
sudo apt-get install -y ca-certificates curl gnupg lsb-release
sudo mkdir -p /etc/apt/keyrings
```

### DNS Troubleshooting (if needed)

If you encounter DNS resolution errors (e.g., "Could not resolve host"), verify your DNS:

```bash
# Check DNS resolution
nslookup download.docker.com

# If it fails, temporarily use Google DNS
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf > /dev/null
echo "nameserver 8.8.4.4" | sudo tee -a /etc/resolv.conf > /dev/null

# For permanent DNS fix on systemd-based systems:
# sudo nano /etc/systemd/resolved.conf
# Add: DNS=8.8.8.8 8.8.4.4
# Then: sudo systemctl restart systemd-resolved
```

Add Docker GPG key:

```bash
sudo rm -f /etc/apt/keyrings/docker.gpg
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

**Verify the key was downloaded successfully:**

```bash
ls -la /etc/apt/keyrings/docker.gpg
```

If the above command fails with DNS errors, try using an alternative mirror or wait a few moments:

```bash
# Alternative: download with retry
curl --retry 5 --retry-delay 3 -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg

# Or use wget as an alternative
wget -qO- https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

Add Docker repository:

```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | \
sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

Update & install Docker + Compose plugin:

```bash
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

Add user to Docker group:

```bash
sudo usermod -aG docker $USER
```

👉 **Important:** Log out & log back in (or `newgrp docker`) to apply changes.

---

## 3. Docker Compose (V2)

Alohida o‘rnatish shart emas — Compose hozir Docker ichida **plugin** sifatida mavjud.

Tekshirish:

```bash
docker compose version
```

---

## 4. Configure SSH Access to GitHub

Install OpenSSH client:

```bash
sudo apt install -y openssh-client
```

Generate SSH key:

```bash
mkdir -p ~/.ssh/github
ssh-keygen -t rsa -b 4096 -C "you@email.com" -f ~/.ssh/github/id_rsa -N ""
```

Copy public key:

```bash
cat ~/.ssh/github/id_rsa.pub
```

Add this key to
**GitHub → Settings → SSH and GPG keys → New SSH Key**

Configure SSH:

```bash
nano ~/.ssh/config
```

Add:

```
Host github.com
    IdentityFile ~/.ssh/github/id_rsa
```

Test connection:

```bash
ssh -T git@github.com
```

GitHub shu xabarni chiqarishi kerak:
`You've successfully authenticated, but GitHub does not provide shell access.`

---

## 5. Clone & Deploy Project

Clone repository:

```bash
git clone git@github.com:<your-username>/<your-repo>.git
cd <your-repo>
```

Deploy:

```bash
docker compose -f production.yml up --build -d
```

Check running containers:

```bash
docker ps
```

---

## 6. Database Management

### Backup (dump):

```bash
docker exec -t <db-container> pg_dump -U <db-user> -d <db-name> \
> dump_$(date +%d-%m-%Y_%H-%M-%S).sql
```

### Copy dump from server → local:

```bash
scp -i <your-key.pem> ubuntu@<SERVER_IP>:/path/to/dump.sql /local/path/
```

### Restore database:

```bash
cat dump.sql | docker exec -i <db-container> psql -U <db-user> -d <db-name>
```

# 🔥 7. Firewall (UFW) Setup

Enable basic firewall, only SSH + HTTP + HTTPS ochiq.

Install UFW (ko‘p holatda oldindan bor):

```bash
sudo apt install ufw -y
```

Allow SSH (albatta birinchi!):

```bash
sudo ufw allow OpenSSH
```

Allow HTTP/HTTPS:

```bash
sudo ufw allow 80
sudo ufw allow 443
```

Enable firewall:

```bash
sudo ufw enable
```

Status:

```bash
sudo ufw status verbose
```

Agar custom portlarda backend yoki admin panel bo‘lsa — qo‘shib qo‘yasiz.

---

# 🕸 8. NGINX Reverse Proxy (SSL-ready)

Install NGINX:

```bash
sudo apt install nginx -y
```

Start:

```bash
sudo systemctl enable nginx
sudo systemctl start nginx
```

---

## 8.1 Reverse Proxy Config (Docker Compose service nomiga yo‘naltirilgan)

Misol uchun backend `app` nomli container’da va `8000` portda ishlayotgan bo‘lsa.

Create config:

```bash
sudo nano /etc/nginx/sites-available/project.conf
```

Config:

```
server {
    listen 80;
    server_name YOUR_DOMAIN.COM;

    client_max_body_size 50M;

    location / {
        proxy_pass http://127.0.0.1:8000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Enable:

```bash
sudo ln -s /etc/nginx/sites-available/project.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

---

## 8.2 HTTPS (Let's Encrypt SSL)

Install Certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

Issue certificate:

```bash
sudo certbot --nginx -d YOUR_DOMAIN.COM
```

Auto-renew test:

```bash
sudo certbot renew --dry-run
```

SSL shu bilan tayyor.

---

# 🔐 9. Fail2Ban Setup (SSH brute-force himoya)

Install:

```bash
sudo apt install fail2ban -y
```

Default config qimirlatilmaydi. Bizga local override yetadi.

Create local jail config:

```bash
sudo nano /etc/fail2ban/jail.local
```

Paste:

```
[DEFAULT]
bantime = 1h
findtime = 10m
maxretry = 5
ignoreip = 127.0.0.1/8

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 5
```

Enable & restart:

```bash
sudo systemctl enable fail2ban
sudo systemctl restart fail2ban
```

Check status:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

---

# 🎯 10. Optional: NGINX rate limiting (DDoS / floodga qarshi)

Configga qo‘shish (site config ichida, server{} yuqorisida):

```
limit_req_zone $binary_remote_addr zone=mylimit:10m rate=10r/s;
```

Server ichiga qo‘shish:

```
location / {
    limit_req zone=mylimit burst=20 nodelay;
    proxy_pass http://127.0.0.1:8000;
    ...
}
```

---

[Telegram](https://t.me/davronbek_dev)
