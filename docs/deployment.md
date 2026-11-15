# Деплой на сервер

Предлагаемый сценарий для VPS с Ubuntu и systemd. Это шаблон самостоятельного размещения: готовых серверных конфигураций в репозитории нет. Перед публичным доступом необходимо исправить проблемы из [known-issues.md](known-issues.md), особенно пароли, права на объекты, CSRF и зачёт отчётов.

```text
Интернет → HTTPS/nginx → 127.0.0.1:8000 Gunicorn → Flask → SQLite + uploads
```

## 0. Что нужно

- VPS с доступом по SSH и Python, совместимым с выбранными зависимостями.
- Домен, указывающий на IP сервера, для HTTPS.
- Постоянный диск для app/instance и app/static/uploads.
- Исправленная версия приложения. Установка Python зависит от образа ОС; ниже команда python3 должна указывать на проверенный интерпретатор.

В примерах пользователь — deploy, каталог — `/srv/site-of-game-tasks`, домен — example.com. Подставьте свои значения. Gunicorn здесь запускается одним worker для простого учебного сценария с SQLite.

## 1. Пользователь для работы (не root)

Используйте пользователя deploy с SSH-ключом и sudo. Если его ещё нет, администратор сервера создаёт его:

```bash
sudo adduser deploy
sudo usermod -aG sudo deploy
```

Настройте authorized_keys и проверьте вход `ssh deploy@SERVER_IP` в отдельном терминале до изменения SSH-настроек.

## 2. SSH только по ключам

При необходимости настройте PasswordAuthentication no и PermitRootLogin no в конфигурации sshd. Проверьте `sudo sshd -t` и эффективные параметры `sudo sshd -T` перед перезапуском. Сохраняйте текущую SSH-сессию до успешного повторного входа: расположение включаемых файлов зависит от образа сервера.

## 3. Firewall

```bash
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
sudo ufw status
```

Если SSH использует другой порт, разрешите его до включения firewall. Gunicorn будет слушать только loopback; порт 8000 наружу не открывается.

## 4. Python и WSGI-сервер

На сервере нужны git, модуль venv для выбранного Python, nginx и Gunicorn в виртуальном окружении. Контейнеризация в проекте не подготовлена.

```bash
sudo apt update
sudo apt install -y git python3-venv nginx
sudo install -d -o deploy -g deploy /srv/site-of-game-tasks
```

Не обновляйте старый Flask независимо от остального стека без проверки совместимости. Предложенный локальный набор указан в [development.md](development.md).

## 5. Код на сервере

Под пользователем deploy:

```bash
git clone https://github.com/hurakot/site-of-game-tasks.git /srv/site-of-game-tasks
cd /srv/site-of-game-tasks
python3 -m venv .venv
.venv/bin/python -m pip install -r requirements.txt
.venv/bin/python -m pip install Flask-SQLAlchemy==3.0.5 SQLAlchemy==2.0.36 Flask-Login==0.6.3 gunicorn
.venv/bin/python -m pip check
```

Для воспроизводимого размещения выбранную версию Gunicorn также следует закрепить в зависимостях подготовленной серверной версии.

Репозиторий содержит учебные базы и фотографии. Для чистого сервиса используйте отдельную копию развёртывания с подготовленной пустой базой или мигрированными данными. Не подменяйте рабочую базу файлом из git при обновлении.

## 6. Настройки и секреты

В app.py сначала замените присваивание SECRET_KEY на `os.environ['SECRET_KEY']`, как описано в [configuration.md](configuration.md). Один EnvironmentFile без этого изменения не работает. Зафиксируйте однозначный путь SQLite и убедитесь, что процесс может писать в каталог базы и uploads.

```bash
sudo install -m 600 /dev/null /etc/site-of-game-tasks.env
python3 -c "import secrets; print(secrets.token_hex(32))"
sudoedit /etc/site-of-game-tasks.env
```

Содержимое файла: `SECRET_KEY=сгенерированное_значение`. Это файл переменных systemd, не автоматически читаемый приложением .env. Остальные изменения безопасности должны быть выполнены отдельно.

## 7. Запуск

Создайте `/etc/systemd/system/site-of-game-tasks.service`:

```ini
[Unit]
Description=Site of Game Tasks
After=network.target

[Service]
User=deploy
Group=deploy
WorkingDirectory=/srv/site-of-game-tasks/app
EnvironmentFile=/etc/site-of-game-tasks.env
ExecStart=/srv/site-of-game-tasks/.venv/bin/gunicorn --workers 1 --bind 127.0.0.1:8000 --access-logfile - --error-logfile - app:application
Restart=on-failure
UMask=0077

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now site-of-game-tasks
sudo systemctl status site-of-game-tasks
curl -I http://127.0.0.1:8000/
```

Из каталога app импорт `app:application` находит app.py и models.py. Блок debug-запуска не выполняется. Отдельного health endpoint нет: `/` обращается к базе и рендерит каталог.

Логи: `sudo journalctl -u site-of-game-tasks -n 100 --no-pager`. Для закрытой проверки можно использовать SSH-туннель: `ssh -L 8000:127.0.0.1:8000 deploy@SERVER_IP`.

## 8. nginx — доступ из интернета

Создайте `/etc/nginx/sites-available/site-of-game-tasks`:

```nginx
server {
    listen 80;
    server_name example.com;
    server_tokens off;
    client_max_body_size 16m;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

16m — пример ограничения всего запроса, включая увеличение данных Base64. Согласуйте его с MAX_CONTENT_LENGTH приложения. Передача proxy-заголовков сама по себе не настраивает доверие к ним в Flask; если понадобится генерация внешних HTTPS-ссылок, отдельно настройте обработку только доверенного прокси.

```bash
sudo ln -s /etc/nginx/sites-available/site-of-game-tasks /etc/nginx/sites-enabled/site-of-game-tasks
sudo nginx -t
sudo systemctl reload nginx
```

Не удаляйте конфигурации других сайтов. Проверьте, что server_name не конфликтует с существующими блоками. До HTTPS не отправляйте пароли через публичное соединение.

## 9. HTTPS

После настройки DNS можно выпустить сертификат через Certbot:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com
sudo certbot renew --dry-run
```

После включения HTTPS задайте в приложении SESSION_COOKIE_SECURE=True, SESSION_COOKIE_HTTPONLY=True и подходящий SESSION_COOKIE_SAMESITE. Это предлагаемые изменения, в текущем коде они явно не заданы.

## 10. Обновление приложения

Сначала сделайте резервную копию. Если исходники и изменяемые данные остаются в одном checkout, убедитесь, что обновление не заменяет отслеживаемые Course.db и uploads. Надёжнее заранее вынести данные из checkout и настроить абсолютные пути.

После проверки diff и миграций:

```bash
cd /srv/site-of-game-tasks
git pull --ff-only
.venv/bin/python -m pip install -r requirements.txt
sudo systemctl restart site-of-game-tasks
curl -I http://127.0.0.1:8000/
```

Недостающие зависимости должны быть закреплены в подготовленной версии. create_all не заменяет миграции; простая замена исходников не обновляет схему.

## 11. Перезагрузка сервера

enable включает автозапуск systemd. После плановой перезагрузки проверьте состояние site-of-game-tasks и nginx, затем главную страницу и вход. Данные должны сохраняться на постоянном диске.

## 12. Бэкап БД и изображений

Простой согласованный бэкап требует краткой остановки приложения. Сначала проверьте реальный путь базы. Ниже путь соответствует Flask-SQLAlchemy 3.x; команды выполняются из корня развёртывания. Для каждого снимка используйте новый каталог.

```bash
mkdir -p /home/deploy/backups/snapshot-01
chmod 700 /home/deploy/backups/snapshot-01
sudo systemctl stop site-of-game-tasks
```

Создать согласованную SQLite-копию стандартным backup API:

```bash
.venv/bin/python - <<'PY'
import sqlite3
source = sqlite3.connect('file:app/instance/Course.db?mode=ro', uri=True)
target = sqlite3.connect('/home/deploy/backups/snapshot-01/Course.db')
try:
    source.backup(target)
finally:
    target.close()
    source.close()
PY
cp -a app/static/uploads /home/deploy/backups/snapshot-01/uploads
sudo systemctl start site-of-game-tasks
```

Если копирование упало, выясните причину и отдельно запустите сервис. Сохраните вместе со снимком версию исходников и настройки без публикации секретов. Перенесите копию на другой носитель по SSH.

Восстановление: остановить сервис, сохранить текущие данные отдельным снимком, проверить целостность копии, заменить рабочую базу и каталог uploads согласованной парой, проверить владельца/права, запустить сервис. Для восстановления в существующую SQLite-базу используйте backup API в обратном направлении, а не копирование поверх открытой базы с возможными WAL-файлами. Процедуру сначала проверить на отдельном экземпляре.

Бэкапы содержат открытые пароли и пользовательские изображения; доступ к ним должен быть ограничен. Автоматического расписания резервного копирования проект не содержит.

## Частые проблемы

| Симптом | Проверка |
|---|---|
| 502 | systemctl status и journalctl: запущен ли Gunicorn |
| ModuleNotFoundError | Окружение, недостающие зависимости, WorkingDirectory=.../app |
| Пустая/не та база | db.engine.url.database и версия Flask-SQLAlchemy |
| readonly database | Права на файл и каталог SQLite, пользователя сервиса |
| 413 | Лимиты nginx и приложения, увеличение размера Base64 |
| 500 на первом зачёте | Известная ошибка деления на ноль; исправить обработчик |
| TemplateNotFound upload_image.html | Незавершённый object upload |

## Проверка перед публикацией

Проверьте исправление известных уязвимостей, отсутствие debug-сервера, внешний секрет, HTTPS, loopback-привязку Gunicorn, постоянное хранение данных и успешное пробное восстановление резервной копии. Эти меры требуют подготовки кода и инфраструктуры; текущая учебная версия их автоматически не обеспечивает.
