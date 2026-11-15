# Разработка и тесты

## Локальный запуск без Docker

Версия Python не закреплена. Для старого стека предлагается отдельное окружение Python 3.10; совместимость всех версий не проверена.

Из корня проекта:

```bash
python3.10 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
python -m pip install Flask-SQLAlchemy==3.0.5 SQLAlchemy==2.0.36 Flask-Login==0.6.3
cd app
python app.py
```

Открыть http://127.0.0.1:5000/, остановить Ctrl+C. На Windows окружение активируется через `.venv\Scripts\Activate.ps1`. Запуск из app учитывает импорт `from models import ...`.

### Подготовка учебных данных

На пустой базе нет администратора и сертификатов. Следующий пример выполнить из app на локальной учебной базе. Пароль запрашивается интерактивно и сохраняется открытым текстом, как ожидает текущий вход.

```bash
python - <<'PY'
from datetime import date
from getpass import getpass
from app import app, db, User, Administrator, Certificate

with app.app_context():
    if User.query.filter_by(username='local-admin').first() is None:
        user = User(username='local-admin',
                    password=getpass('Учебный пароль администратора: '),
                    registration_date=date.today())
        db.session.add(user)
        db.session.flush()
        db.session.add(Administrator(id_administrator=user.id))
    if Certificate.query.filter_by(certificate_code='LOCAL-DEMO-001').first() is None:
        db.session.add(Certificate(certificate_code='LOCAL-DEMO-001',
                                  registration_date_certificate=date.today()))
    db.session.commit()
PY
```

Зарегистрировать наёмника с cert `LOCAL-DEMO-001` через `/create_mercenary/`. Для следующего пользователя нужен другой сертификат. Скрипт не меняет существующего администратора и не освобождает занятый сертификат. Представитель регистрируется через `/create_member/` с учебной парой из [configuration.md](configuration.md).

Работайте с копией проекта и данных: приложение подключается к базе уже при импорте. JustForTest.py использует другой файл и другую схему, его запускать для подготовки приложения не нужно.

## Локальный запуск в Docker

Dockerfile и docker-compose.yaml отсутствуют; команды Docker из PET-task-manager здесь не применимы. Контейнеризация потребует отдельного WSGI-запуска и постоянных томов для SQLite/uploads.

## Тесты

Автотестов, pytest.ini и requirements-dev.txt нет. Ниже — сценарий ручной проверки на одноразовых данных, а не отчёт об успешном тестировании.

| Шаг | Проверка |
|---|---|
| Гость | Каталог показывает Active-задания, рейтинг — Alive-наёмников |
| Представитель | Верная пара организации/кода создаёт профиль; повтор имени отвергается |
| Вход | Верные данные создают сессию, неверные возвращают на форму |
| Создание задания | Задание получает Active, организацию представителя и дату создания |
| Наёмник | Свободный существующий сертификат принимается; повторное использование отклоняется |
| Принятие | Задание получает in_process, ID исполнителя и doing_task |
| Отчёт | Создаются Active-отчёт, текст и файлы app/static/uploads |
| Представитель | Отчёт виден в кабинете соответствующей организации |
| Отклонение | Отчёт получает Rejected, задание остаётся in_process |
| Администратор | Просмотр/изменение таблиц, удаление только одноразовых записей |
| Выход | POST /goout завершает сессию |

Первый зачёт сейчас может завершиться ZeroDivisionError после фиксации Done. Повторять зачёт вслепую нельзя: рейтинг может начислиться повторно. Проверки чужих объектов должны выявлять текущую уязвимость — см. [known-issues.md](known-issues.md).

Приоритет будущих автотестов: права на чужие объекты, первый/повторный зачёт, область изменения отчётов, конкурентное принятие задания, сертификаты, загрузки. Нужна изолированная временная база и настройка приложения до инициализации ORM.

## Зависимости

requirements.txt содержит Flask, Werkzeug, Jinja2, MarkupSafe, itsdangerous, click, Faker, python-dateutil и six. Импортируемые Flask-SQLAlchemy, SQLAlchemy и Flask-Login отсутствуют. Предложенные выше версии — набор для воспроизведения, а не исходный lock-файл или результат CI.

```bash
python -m pip check
```

Добавляя пакет, закрепите проверенную версию и проверьте установку в чистом окружении. Полный pip freeze из постороннего окружения не нужен.

## Полезные команды

Синтаксическая проверка из корня без запуска приложения:

```bash
python - <<'PY'
import ast
from pathlib import Path
for path in Path('app').glob('*.py'):
    ast.parse(path.read_text(), filename=str(path))
    print(path)
PY
```

WSGI-запуск из app после отдельной установки Gunicorn:

```bash
gunicorn --bind 127.0.0.1:8000 app:application
```

Gunicorn не входит в requirements.txt. Серверная настройка — в [deployment.md](deployment.md).
