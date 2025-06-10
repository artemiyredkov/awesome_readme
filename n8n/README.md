# 🚀 Разворачиваем n8n + PostgreSQL (Docker) + Nginx + SSL

## 📋 Что мы сделаем
- Развернем n8n на собственном сервере, доступный по `n8n.example.com`
- Развернем PostgreSQL
	- Доступ из n8n по внутренней сети
	- Доступ через DBeaver (используя SSL сертификаты)
- Nginx как обратный прокси
- HTTPS-сертификат от Let's Encrypt (с автообновлением)

## 📋 Предварительные требования
- VPS с Ubuntu 20+ (https://beget.com/ru/vps)
- Домен (например: `n8n.example.com`)
- DNS A-запись домена указывает на `n8n.example.com`
- SSH-доступ

## 🎯 Структура проекта
```bash
~/n8n_project/
│
├── .env
├── docker-compose.yml
│
├── n8n_data/
│
├── postgres_data/
├── postgres_ssl/
│   ├── server.crt
│   └── server.key
```

## 1 Создание пользователя
Заходим под ssh root@<your_server_ip> и создаем пользователя
```bash
adduser <your_username>
usermod -aG sudo <your_username>
ssh-copy-id <your_username>@<your_server_ip>
ssh <your_username>@<your_server_ip>
exit
```

## 2 Установка Docker и Docker Compose
Заходим под ssh root@<your_server_ip> и устанавливаем Docker
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sh get-docker.sh
usermod -aG docker <your_username>
apt install docker-compose -y
exit
```

## 3 Подготовка проекта

### 3.1 Возвращаемся под своим пользователем
```bash
ssh <your_username>@<your_server_ip>
```

### 3.2 Создаем папки
```bash
mkdir ~/n8n_project && cd ~/n8n_project
mkdir postgres_data postgres_ssl n8n_data
```

### 3.3 Создаем .env и docker-compose.yml

Генерация ENCRYPTION_KEY: ```openssl rand -hex 32 ```

nano .env

```bash
POSTGRES_USER=artemiiredkov
POSTGRES_PASSWORD=<your_password>
POSTGRES_DB=n8n

N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=<admin_password>
N8N_ENCRYPTION_KEY=<ENCRYPTION_KEY>
N8N_HOST=localhost
N8N_PORT=5678
WEBHOOK_URL=https://n8n.example.com/
WEBHOOK_TUNNEL_URL=
N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=true
TZ=Europe/Moscow
```

nano docker-compose.yml
```bash
yaml

version: '3.8'

services:
  postgres:
    image: postgres:16
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-net

  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=${N8N_PORT}
      - WEBHOOK_URL=${WEBHOOK_URL}
      - WEBHOOK_TUNNEL_URL=${WEBHOOK_TUNNEL_URL}
      - N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE=${N8N_COMMUNITY_PACKAGES_ALLOW_TOOL_USAGE}
      - TZ=${TZ}
    volumes:
      - ./n8n_data:/home/node/.n8n
    depends_on:
      - postgres
    networks:
      - n8n-net

networks:
  n8n-net:
    driver: bridge
```

### 3.4 🚀 Первый запуск
```bash
docker compose up -d
```


## 🔒 4 SSL-сертификаты для PostgreSQL

### 4.1 Генерация SSL-сертификатов для PostgreSQL
```bash
openssl req -new -x509 -days 365 -nodes \
  -out ./postgres_ssl/server.crt \
  -keyout ./postgres_ssl/server.key \
  -subj "/CN=postgres_data"
```  

```bash
chmod 600 ./postgres_ssl/server.key
chmod 644 ./postgres_ssl/server.crt
```

### 4.2 Копирование сертификатов внутрь контейнера:

```bash
docker cp ./postgres_ssl/server.crt postgres:/var/lib/postgresql/data/server.crt
docker cp ./postgres_ssl/server.key postgres:/var/lib/postgresql/data/server.key
```

### 4.3 Выставляем права внутри контейнера:
```bash
docker exec -it postgres bash
chown postgres:postgres /var/lib/postgresql/data/server.*
chmod 600 /var/lib/postgresql/data/server.key
chmod 644 /var/lib/postgresql/data/server.crt
```

### 4.4 Включаем SSL в PostgreSQL 

Добавляем или раскоментируем строчки :
nano ./postgres_data/postgresql.conf
```bash
ssl = on
ssl_cert_file = 'server.crt'
ssl_key_file = 'server.key'
```

### 4.5 Перезапускаем postgres:
```bash
docker compose restart postgres
```

## 5 Установка Nginx + Let's Encrypt

### 5.1 Устанавливаем nginx:

```bash
sudo apt install nginx -y
```

### 5.2 Устанавливаем certbot:

```bash
sudo apt install certbot python3-certbot-nginx -y
```

### 5.3 Получаем сертификат:
```bash
sudo certbot --nginx -d n8n.example.com
```

## 🔧 6 Конфиг Nginx для n8n
sudo nano /etc/nginx/sites-available/n8n
```bash
server {
    listen 80;
    server_name n8n.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name n8n.example.com;

    ssl_certificate /etc/letsencrypt/live/n8n.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/n8n.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:5678/;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 7 Активируем сайт:

```bash
sudo ln -s /etc/nginx/sites-available/n8n /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

## 📦 Готово!

- n8n работает на https://n8n.ваш-домен.ru
- Есть HTTPS и автоматическое продление
- Все данные сохраняются в ~/n8n_data
- Подключение PostgreSQL из n8n
```
	Host: postgres
	Port: 5432
	SSL: отключен
	Подключение работает через сеть n8n-net
```
- Подключение PostgreSQL из DBeaver (с SSL)
```
	Включаем SSL в настройках подключения
	Указываем:
	SSL режим: require
	Клиентский сертификат: не требуется
	Клиентский ключ: не требуется
	Корневой сертификат: server.crt
```

## 🧠 Полезные команды

- Проверка nginx: `sudo nginx -t`
- Перезапуск nginx: `sudo systemctl reload nginx`
- Логи n8n: `docker logs -f n8n`
- Стоп и удаление коконтейнера
	- `docker stop n8n`
	- `docker rm n8n`