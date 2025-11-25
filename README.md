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

Add Docker GPG key:

```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
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

