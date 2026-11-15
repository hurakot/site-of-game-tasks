# Site of Game Tasks

Веб-приложение для игровых заданий: представители организаций публикуют задания, наёмники выполняют их и отправляют отчёты с изображениями. Администратор управляет заданиями и учётными записями.

## Идея проекта

Курсовой проект по дисциплине «Разработка веб-приложений» (2 курс, 4 семестр):

- регистрация представителей и наёмников по сертификатам;
- вход и сессии через Flask-Login;
- каталог заданий с пагинацией и скачиванием описания в TXT;
- кабинеты ролей `admin`, `member`, `mercenary`;
- создание и принятие заданий, отправка и проверка отчётов;
- рейтинг наёмников и административные таблицы.

Это учебная версия с известными ошибками — см. [ограничения](docs/known-issues.md).

## Стек технологий

| Область | Технологии |
|---|---|
| Язык | Python, версия не закреплена |
| Веб-фреймворк | Flask 2.0.3, Werkzeug 2.0.3 |
| ORM | Flask-SQLAlchemy, SQLAlchemy |
| База данных | SQLite, Course.db |
| Авторизация | Flask-Login, cookie-сессии |
| Интерфейс | Jinja2 3.0.3, HTML, CSS, JavaScript, Bootstrap через CDN |
| Изображения | Base64 в форме → app/static/uploads/ |
| Дополнительно | Faker 13.3.2, импортирован в приложении |

**Деплой:** Dockerfile, Docker Compose и конфигурации хостинга отсутствуют. Инструкция самостоятельного размещения приведена в [deployment.md](docs/deployment.md).

## Документация

| Файл | Содержание |
|---|---|
| [docs/architecture.md](docs/architecture.md) | Структура, назначение файлов, слои, данные, путь запроса |
| [docs/api.md](docs/api.md) | Все маршруты, формы, роли, ответы, curl |
| [docs/configuration.md](docs/configuration.md) | Настройки, пути базы и изображений, секреты |
| [docs/development.md](docs/development.md) | Запуск, учебные данные, проверки и зависимости |
| [docs/deployment.md](docs/deployment.md) | VPS, Gunicorn, systemd, nginx, HTTPS, бэкапы |
| [docs/known-issues.md](docs/known-issues.md) | Ошибки, ограничения и планы |

## Быстрый старт (локально, без Docker)

Для старого стека можно использовать Python 3.10. Это предлагаемый вариант совместимости, а не проверенная матрица версий.

```bash
git clone https://github.com/hurakot/site-of-game-tasks.git Site-of-game-tasks
cd Site-of-game-tasks
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
# Используются кодом, но отсутствуют в requirements.txt:
python -m pip install Flask-SQLAlchemy==3.0.5 SQLAlchemy==2.0.36 Flask-Login==0.6.3
cd app
python app.py
```

Открыть http://127.0.0.1:5000/. Отладочный сервер предназначен для локальной разработки. Остановка — `Ctrl+C`.

1. Создать представителя на `/create_member/` — учебные организации и коды в [configuration.md](docs/configuration.md).
2. Войти на `/authentication/` и создать задание на `/create_task/`.
3. Подготовить свободный сертификат по [development.md](docs/development.md), зарегистрировать наёмника.
4. В отдельной сессии браузера войти наёмником, принять задание и отправить отчёт.

Таблицы создаются при импорте. Начальные пользователи и сертификаты автоматически не добавляются. Для Flask-SQLAlchemy 3.x рабочая база — `app/instance/Course.db`; корневой `instance/Course.db` — отдельный файл.

## Тесты

Автотестов нет. `app/JustForTest.py` — отдельный эксперимент с SQLite. Ручные проверки описаны в [development.md](docs/development.md).

## Деплой

Готовой конфигурации Render нет. Перед публикацией необходимо исправить проблемы безопасности и зачёта отчётов из [known-issues.md](docs/known-issues.md).

## Запуск на собственном сервере (VPS)

Предлагаемая схема: браузер → HTTPS/nginx → Gunicorn на `127.0.0.1:8000` → Flask → SQLite и uploads на постоянном диске. Команда из каталога `app`: `gunicorn --bind 127.0.0.1:8000 app:application`.

Подготовка сервера и резервное копирование — в [deployment.md](docs/deployment.md).

## Безопасность

- Пароли сейчас сохраняются открытым текстом; ключ сессии и коды организаций зашиты в код.
- Проверка роли не обеспечивает проверку владельца объекта; часть изменений доступна через GET, CSRF-защиты нет.
- Изображения uploads публичны; проверка формата и общий лимит запроса не настроены.
- Используйте вымышленные данные и отдельные учебные пароли. План исправлений — в [known-issues.md](docs/known-issues.md).
