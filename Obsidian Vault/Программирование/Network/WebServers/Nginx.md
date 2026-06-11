## **Nginx как REST API для раздачи файлов**  
*Цель: `GET /api/v1/files/photo.jpg` → отдаёт файл из локальной директории*

---

### Установка и базовая проверка

```bash
# Ubuntu/Debian
sudo apt update && sudo apt install nginx -y

# Запуск
sudo systemctl enable --now nginx

# Проверка
curl -I http://localhost
# → HTTP/1.1 200 OK
```

---

### Подготовка директории с файлами

```bash
# Создаём папку (лучше НЕ в /home/ — избегаем проблем с правами)
sudo mkdir -p /var/www/media-files

# Копируем/создаём тестовые файлы
sudo cp ~/resources/media/*.jpg /var/www/media-files/
sudo chown -R www-data:www-data /var/www/media-files   # ← рекомендуется!
sudo chmod -R 755 /var/www/media-files
```

> 💡 Почему `/var/www/media-files`, а не домашняя папка?  
> — У `www-data` (пользователь nginx) есть права по умолчанию → никаких `403/404` из-за `chmod`.

---

### Конфигурация nginx: REST-like API

Создайте файл конфига:
```bash
sudo nano /etc/nginx/sites-available/media-api
```

Вставьте:

```nginx
server {
    listen 8000; # port
    server_name localhost; # ip

    # REST-style endpoint: /api/v1/files/<filename>
    location /api/v1/files/ {
        alias /var/www/media-files/;

        # Поддержка Range-запросов (для видео/аудио/перемотки)
        # Включено по умолчанию, но явно укажем для ясности
        add_header Accept-Ranges bytes;

        # Кэширование: 1 час для клиентов, 1 день для прокси
        expires 1h;
        add_header Cache-Control "public, max-age=3600";

        # Запретить скрытые файлы (.env, .git и т.д.)
        location ~ /\. {
            deny all;
            return 404;
        }

        # Автоиндекс (ТОЛЬКО для отладки! В продакшене — убрать)
        # autoindex on;
    }

    # Health-check endpoint
    location /health {
        return 200 '{"status":"ok","service":"media-api"}';
        add_header Content-Type application/json;
    }
}
```

---

### 🔗 4. Активация конфига

```bash
# Включаем конфиг
sudo ln -sf /etc/nginx/sites-available/media-api /etc/nginx/sites-enabled/

# Проверяем синтаксис
sudo nginx -t
# → должен быть: syntax is ok, test is successful

# Перезагружаем
sudo systemctl reload nginx
```

---

### Тестирование

```bash
# 1. Проверить health-check
curl http://localhost:8000/health
# → {"status":"ok","service":"media-api"}

# 2. Скачать файл
curl -I http://localhost:8000/api/v1/files/photo.jpg
```

Ожидаемый ответ:
```
HTTP/1.1 200 OK
Server: nginx
Content-Type: image/jpeg
Content-Length: 555181
Accept-Ranges: bytes   ← ← ← критично для видео!
Cache-Control: public, max-age=3600
...
```

Открыть в браузере:  
→ `http://localhost:8000/api/v1/files/photo.jpg`

---

## Продвинутые фичи (по необходимости)

### A. **Поддержка CORS** (если API используется из браузера)
Добавьте в `location /api/v1/files/`:
```nginx
add_header 'Access-Control-Allow-Origin' '*' always;
add_header 'Access-Control-Allow-Methods' 'GET, OPTIONS' always;
add_header 'Access-Control-Allow-Headers' 'DNT,User-Agent,X-Requested-With,If-Modified-Since,Cache-Control,Content-Type,Range' always;
add_header 'Access-Control-Expose-Headers' 'Content-Length,Content-Range' always;

# Ответ на OPTIONS (preflight)
if ($request_method = 'OPTIONS') {
    add_header 'Access-Control-Max-Age' 1728000;
    add_header 'Content-Type' 'text/plain; charset=utf-8';
    add_header 'Content-Length' 0;
    return 204;
}
```

---

### B. **Ограничение по типам файлов** (безопасность)
Разрешить только изображения и видео:
```nginx
location ~ ^/api/v1/files/.*\.(jpg|jpeg|png|gif|mp4|webm|mp3)$ {
    alias /var/www/media-files/;
    # ... остальные настройки
}

# Запретить всё остальное
location /api/v1/files/ {
    return 403 "Forbidden file type";
}
```

---

### C. **Подпись URL (expiring links)** — через `secure_link`
Генерируйте ссылки вида:  
`/api/v1/files/photo.jpg?md5=abc123&expires=1732200000`

Конфиг:
```nginx
location /api/v1/files/ {
    alias /var/www/media-files/;
    secure_link $arg_md5,$arg_expires;
    secure_link_md5 "$secure_link_expires$uri$remote_addr secret_key";

    if ($secure_link = "") { return 403; }
    if ($secure_link = "0") { return 410; }  # expired
}
```
→ Подробнее: [nginx secure_link module](http://nginx.org/en/docs/http/ngx_http_secure_link_module.html)

---

### D. **Логирование запросов к файлам**
```nginx
log_format media_api '$remote_addr - $remote_user [$time_local] '
                     '"$request" $status $body_bytes_sent '
                     '"$http_referer" "$http_user_agent" '
                     'rt=$request_time uct="$upstream_connect_time"';

access_log /var/log/nginx/media_api.log media_api;
```

---

## Производительность: рекомендуемые настройки в `nginx.conf`

В секции `http { ... }`:
```nginx
sendfile on;
tcp_nopush on;
tcp_nodelay on;

# Для больших файлов
client_body_buffer_size 128k;
client_max_body_size 0;   # или 10G

# Таймауты
send_timeout 10m;
keepalive_timeout 65;

# Кэш метаданных (ускоряет stat())
open_file_cache max=1000 inactive=20s;
open_file_cache_valid 30s;
open_file_cache_min_uses 2;
open_file_cache_errors on;
```

---

## Безопасность: checklist

| Рекомендация | Как |
|-------------|-----|
| ❌ Не используйте домашние папки (`/home/user/...`) | → Используйте `/var/www/...` + `chown www-data` |
| ✅ Ограничьте типы файлов | `location ~ \.(jpg\|png\|mp4)$` |
| ✅ Запретите `.env`, `.git`, `backup.zip` | `location ~ /\.(env\|git\|bak)` { deny all; } |
| ✅ Отключите `autoindex` в продакшене | Удалите `autoindex on;` |
| ✅ Используйте HTTPS | `certbot` + `listen 443 ssl http2;` |

---

## Пример: как интегрировать с бэкендом (auth proxy)

Если файлы должны отдаваться **только авторизованным**:
```nginx
location /api/v1/files/ {
    auth_request /auth;   # → вызывает ваш бэкенд для проверки токена
    alias /var/www/media-files/;
}

location = /auth {
    internal;
    proxy_pass http://backend:8000/auth;
    proxy_pass_request_body off;
    proxy_set_header Content-Length "";
    proxy_set_header X-Original-URI $request_uri;
}
```
→ Бэкенд возвращает `200` (разрешить) или `401/403` (запретить).

---

## Итог:  REST API для файлов

| Эндпоинт | Метод | Описание |
|---------|-------|----------|
| `GET /api/v1/files/photo.jpg` | GET | Отдаёт файл (с поддержкой Range, CORS, кэширования) |
| `GET /health` | GET | Health-check |
| `OPTIONS /api/v1/files/...` | OPTIONS | Preflight для CORS |
