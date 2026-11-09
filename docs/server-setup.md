# Конфигурация виртуальной машины devops-vm
## 1. Параметры машины
(объём памяти, количество ядер, объём диска)
## 2. Сетевые интерфейсы
(тип адаптера, полученный адрес, назначение)
## 3. Правило проброса портов
(порт хостовой системы, порт гостевой системы)
## 4. Учётные записи
(имя, принадлежность к группам, способ аутентификации)
## 5. Служба SSH
(порт, изменённые директивы и их значения, расположение файла конфигурации)
## 6. Правила межсетевого экрана
(политики по умолчанию, перечень правил)
## 7. Снимки состояния
(наименование, момент создания)

## 8. Веб-сервер

- Пакет: `nginx` (`sudo apt install -y nginx`).
- Конфигурация ресурса: `/etc/nginx/sites-available/devops-site`, включена ссылкой в `/etc/nginx/sites-enabled/`; стандартный ресурс `default` отключён (удалена ссылка).
- Каталог ресурса: `/var/www/devops-site`, владелец `devops:devops`, права: каталоги `755`, файлы `644`.
- Сертификат: `/etc/ssl/certs/devops.crt`, права `644`, владелец `root`.
- Закрытый ключ: `/etc/ssl/private/devops.key`, права `600`, владелец `root`.
- Срок действия сертификата: 365 дней, самоподписанный, с полем SAN `DNS:devops.local`.
- Команда формирования сертификата:

```bash
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout /etc/ssl/private/devops.key \
  -out /etc/ssl/certs/devops.crt \
  -subj "/CN=devops.local" \
  -addext "subjectAltName=DNS:devops.local"
```

- Порты 80 и 443 разрешены в UFW (правила из раздела о файрволе); HTTP перенаправляется на HTTPS кодом 301.
- Применение конфигурации: `sudo nginx -t && sudo systemctl reload nginx`.

## 9. Контейнеризация

- Docker Engine установлен из официального репозитория Docker (download.docker.com, Ubuntu 26.04), версия 29.9.2, плагин Compose v5.7.0. Пакет docker.io из репозитория дистрибутива не используется.
- Зеркало реестра: `/etc/docker/daemon.json` с `{"registry-mirrors": ["https://mirror.gcr.io"]}` (прямой доступ к Docker Hub с ВМ нестабилен: таймауты DNS); после изменения `sudo systemctl restart docker`.
- Участники группы docker: devops (группа даёт фактически права root, поэтому включается только учётная запись с правом sudo).
- Проект на сервере: `/home/devops/app` (приложение Flask + PostgreSQL 16, файлы `Dockerfile`, `docker-compose.yml`, `app.py`, `requirements.txt`); доставляется с хоста командой `rsync -avz --delete --exclude=.env --chmod=D755,F644 app/ devops:app/`.
- Файл `.env`: `/home/devops/app/.env`, владелец devops, права 600, содержит случайный пароль БД (формируется на сервере из `.env.example`: `openssl rand -hex 16`), в репозиторий не добавляется.
- Публикуемые порты: приложение `127.0.0.1:5000` → 5000 контейнера (только локальный адрес); порт СУБД 5432 не публикуется, приложение обращается к ней по имени сервиса `db` во внутренней сети `backend` (`app_backend`).
- Именованный том: `db_data` (`app_db_data`), хранит данные PostgreSQL; `docker compose down` его сохраняет, `docker compose down -v` удаляет.
- Обратный прокси-сервер: в блок HTTPS файла `/etc/nginx/sites-available/devops-site` добавлено расположение:

```nginx
location /app/ {
    proxy_pass http://127.0.0.1:5000/;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}
```