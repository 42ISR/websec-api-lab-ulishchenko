https://ulishchenko.kitek-pg.ru/api/marketplace/items

# Лабораторная: Деплой API

## Цель

Собрать API на Express.js в Docker-образ, запустить его через Docker Compose и опубликовать через SWAG по HTTPS на вашем домене:

```text
https://%ваш_сабдомен%.kitek-pg.ru/api/marketplace/
```

Схема работы:

```text
Клиент
  ↓ HTTPS :443
SWAG (nginx + сертификат)
  ↓ HTTP, docker-сеть, имя контейнера
marketplace-backend :3001
```

---

# 1. Проверить исходное состояние

Проверить, что SWAG уже работает:

```bash
docker ps
```

Проверить, что ваш домен открывается по HTTPS:

```bash
curl -I https://%ваш_сабдомен%.kitek-pg.ru
```

---

# 2. Подготовить проект API

Создайте каталог для бэкенда рядом с `docker-compose.yaml`:

```bash
git clone https://github.com/41ISR-2026/webdev-lab8
mv webdev-lab8/backend marketplace-backend
```

И удалите папку webdev-lab8

Структура проекта должна быть такой:

```text
marketplace-backend/
├── Dockerfile
├── .dockerignore
├── package.json
├── package-lock.json
├── index.js
└── database.json
```

Права на файл, чтобы пользователь `node` внутри контейнера мог в него писать:

```bash
sudo chown 1000:1000 database.json
```

---

# 3. Написать Dockerfile

Создайте файл `Dockerfile`:

```bash
nano Dockerfile
```

```dockerfile
FROM node:22-slim

WORKDIR /app

COPY package*.json ./
RUN npm ci --omit=dev

COPY --chown=node:node . .

RUN chown node:node /app
USER node

EXPOSE 3001

CMD ["node", "index.js"]
```

Разбор:

| Инструкция | Зачем |
|---|---|
| `FROM node:22-slim` | Базовый образ с Node.js 22 |
| `WORKDIR /app` | Рабочий каталог внутри образа |
| `COPY package*.json ./` и `RUN npm ci` | Сначала ставим зависимости. Этот слой кэшируется и не пересобирается, пока не изменились `package.json` и lock-файл |
| `COPY --chown=node:node . .` | Копируем код приложения и сразу отдаём файлы пользователю `node` |
| `USER node` | Запускаем приложение не от root |
| `EXPOSE 3001` | Документируем порт приложения (наружу он этим не публикуется) |
| `CMD [...]` | Команда запуска при старте контейнера |

---

# 4. Собрать и проверить образ вручную

Вернитесь в каталог с `docker-compose.yaml`:

```bash
cd ..
```

Собрать образ:

```bash
docker build -t marketplace-backend-test ./marketplace-backend
```

---

# 5. Добавить сервис в Docker Compose

Откройте:

```bash
nano docker-compose.yaml
```

В секцию `services` рядом со SWAG добавьте:

```yaml
    marketplace-backend:
        container_name: marketplace-backend
        build: ./marketplace-backend/.
        restart: unless-stopped
        volumes:
            - ./marketplace-backend/database.json:/app/database.json
        networks:
            - proxy
```

> Важно: SWAG и `marketplace-backend` должны быть в одной docker-сети.

Проверить файл:

```bash
docker compose config
```

---

# 6. Запустить контейнер

Собрать и запустить только новый сервис:

```bash
docker compose up -d --build marketplace-backend
```

Проверить:

```bash
docker compose ps
docker logs marketplace-backend
```

---

# 7. Проверить доступность API из контейнера SWAG

Контейнеры обращаются друг к другу по **имени сервиса**. Проверим, что SWAG видит API:

```bash
docker exec swag curl -s http://marketplace-backend:3001/api/items
```

Если вывод пустой или ошибка `Could not resolve host`, контейнеры в разных сетях. Проверить:

```bash
docker inspect swag --format '{{json .NetworkSettings.Networks}}'
docker inspect marketplace-backend --format '{{json .NetworkSettings.Networks}}'
```

Имя сети должно совпадать.

---

# 8. Настроить проксирование в SWAG

Найдите каталог конфигурации SWAG на хосте. Это источник тома, который смонтирован в `/config`:

Откройте основной конфиг сайта:

```bash
nano %путь_к_config%/nginx/site-confs/default.conf
```

Внутри блока `server` с вашим доменом `%ваш_сабдомен%.kitek-pg.ru` (в блоке с `listen 443 ssl`) добавьте `location`:

```nginx
    location /api/marketplace/ {
        include /config/nginx/proxy.conf;
        include /config/nginx/resolver.conf;

        set $upstream_app marketplace-backend;
        set $upstream_port 3001;
        set $upstream_proto http;

        rewrite ^/api/marketplace/(.*)$ /api/$1 break;
        proxy_pass $upstream_proto://$upstream_app:$upstream_port;
    }
```

Что здесь происходит:

| Строка | Назначение |
|---|---|
| `include .../proxy.conf` | Стандартные заголовки проксирования (`Host`, `X-Forwarded-*` и др.) |
| `include .../resolver.conf` | Docker DNS: имя контейнера резолвится динамически, и nginx не упадёт, если API перезапустится |
| `set $upstream_*` | Адрес контейнера в переменных |
| `rewrite ... break` | Убирает `marketplace/` из пути: `/api/marketplace/items` превращается в `/api/items` |
| `proxy_pass` | Отправляет запрос в контейнер `marketplace-backend:3001` |

> `proxy_pass` указан без пути в конце. Так nginx передаёт бэкенду уже переписанный `rewrite` адрес.

---

# 9. Проверить конфигурацию и перезапустить SWAG

Проверить синтаксис:

```bash
docker exec swag nginx -t
```

Ожидаемый результат:

```text
syntax is ok
test is successful
```

Применить конфигурацию:

```bash
docker exec swag nginx -s reload
```

Если что-то пошло не так, смотрите логи:

```bash
docker logs swag --tail 50
```

---

# 10. Проверить API через домен

Проверить здоровье сервиса:

```bash
curl https://%ваш_сабдомен%.kitek-pg.ru/api/marketplace/items
```
