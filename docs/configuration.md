# Конфигурация

Настройки задаются в `app/app.py` до `db.init_app(app)`. Чтение `.env` и переменных окружения для настроек приложения не реализовано. Создание `.env` само по себе ничего не меняет.

## Переменные

| Настройка | Текущее значение | Назначение |
|---|---|---|
| SECRET_KEY | Жёстко заданный учебный ключ | Подпись сессий и flash, заменить перед публикацией |
| SQLALCHEMY_DATABASE_URI | sqlite:///Course.db | SQLite |
| SQLALCHEMY_TRACK_MODIFICATIONS | False | Отключение уведомлений ORM |
| UPLOAD_FOLDER | os.path.join(app.static_folder, 'uploads') | app/static/uploads |
| MAX_FORM_MEMORY_SIZE | 256 * 1024 * 1024 | Задуман как лимит формы; Flask 2.0.3 не обеспечивает им заявленное ограничение |
| MAX_CONTENT_LENGTH | Не установлен, строка закомментирована | Общий размер запроса не ограничен приложением |
| DATABASE | Course.db | Переменная модуля, не управляет ORM |
| debug | True при python app.py | Отладочный сервер |

Прямой запуск использует `127.0.0.1:5000`. WSGI-запуск не выполняет блок app.run.

## DATABASE_URL: локально и на сервере

DATABASE_URL в коде нет: применяется SQLALCHEMY_DATABASE_URI. Для Flask-SQLAlchemy 3.x относительный SQLite URI разрешается от instance_path: рабочая база — `app/instance/Course.db`. Старые версии расширения разрешали путь иначе. Корневой `instance/Course.db` — отдельная копия, файлы не синхронизируются.

Проверить фактические пути из каталога app:

```bash
python - <<'PY'
from app import app, db
with app.app_context():
    print('instance:', app.instance_path)
    print('database:', db.engine.url.database)
    print('uploads:', app.config['UPLOAD_FOLDER'])
PY
```

Импорт создаёт отсутствующие таблицы и uploads. Для однозначного серверного пути можно изменить URI до инициализации ORM: `sqlite:////srv/site-of-game-tasks/app/instance/Course.db`. Каталог должен существовать и быть доступен на запись сервису. Корневой static/uploads не является app.static_folder.

## Генерация секретов

```bash
python3 -c "import secrets; print(secrets.token_hex(32))"
```

Для серверного запуска предлагается заменить присваивание ключа в app.py:

```python
app.config['SECRET_KEY'] = os.environ['SECRET_KEY']
```

os уже импортирован. Это необходимое изменение исходника, а не существующая функция. После него переменную можно передавать через EnvironmentFile systemd. Отсутствующая переменная остановит запуск; смена ключа аннулирует текущие сессии.

Для лимита запроса отдельно задайте MAX_CONTENT_LENGTH и согласуйте его с nginx. Лимит формы не заменяет проверку количества, формата и размера изображений после декодирования.

## Регистрационные данные

Код разрешает пары `Empire` / `empire123`, `Bird` / `bird123`, `Front` / `front123`. Это публичные учебные значения. Наёмнику нужен существующий свободный certificate_code. Интерфейса выдачи сертификатов и регистрации администратора нет; подготовка описана в [development.md](development.md).

Срок задания (семь дней), пагинация (два задания, пять наёмников), статусы и коды организаций находятся в обработчиках и не настраиваются через окружение.
