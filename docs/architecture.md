# Архитектура

## Структура проекта

```text
Site-of-game-tasks/
├── app/
│   ├── app.py                 # Flask, настройки, маршруты и бизнес-логика
│   ├── models.py              # db и семь ORM-моделей
│   ├── JustForTest.py         # отдельный эксперимент sqlite3
│   ├── instance/Course.db     # база Flask-SQLAlchemy 3.x
│   ├── static/
│   │   ├── base.css
│   │   ├── admin.css
│   │   ├── auth.css
│   │   ├── create_account.css
│   │   ├── CRUD_table.css
│   │   ├── images/            # аватар, иллюстрации, пример публикации
│   │   └── uploads/           # фотографии отчётов
│   └── templates/
│       ├── base.html
│       ├── auth/
│       ├── main_pages/
│       ├── user_pages/
│       ├── sub_user_pages/
│       └── admin_tables/
├── docs/                     # шесть тематических документов
├── instance/Course.db        # отдельная копия базы
├── static/uploads/           # отдельная корневая папка
├── requirements.txt
├── .gitignore
└── README.md
```

В локальной папке также есть поясняющий TXT-файл о курсовом проекте, исключённый через `.gitignore`. `.git/` хранит историю репозитория. Бинарное содержимое баз и изображений здесь не дублируется.

### Назначение файлов

`app.py` создаёт `app` и WSGI-псевдоним `application`, инициализирует SQLAlchemy и Flask-Login. При импорте создаёт отсутствующие таблицы и каталог uploads. Содержит аутентификацию, проверку ролей, маршруты, обработку изображений. Прямой запуск включает debug server.

`models.py` содержит `db`, поля таблиц и связи. Свойство `User.role` проверяет профильные записи в порядке `admin`, `member`, `mercenary`.

`JustForTest.py` создаёт `my_database.db` относительно текущего каталога и таблицу `users(name, age)`. Не импортируется приложением. Повторный запуск может упасть из-за существующей таблицы; это не тестовый набор.

`requirements.txt` закрепляет часть библиотек, но пропускает ORM и Flask-Login. `.gitignore` исключает окружения, кеш Python, `.env` и служебные файлы; SQLite и uploads не исключены.

### Содержимое шаблонов

| Файл внутри templates/ | Назначение |
|---|---|
| `base.html` | Общий каркас, навигация по роли, Bootstrap, блоки дочерних страниц |
| `auth/authentication.html` | Вход |
| `auth/create_member.html` | Регистрация представителя с организацией и кодом |
| `auth/create_mercenary.html` | Регистрация наёмника с Telegram и сертификатом |
| `main_pages/main_page.html` | Карточки заданий, модальные подробности, пагинация |
| `main_pages/rating.html` | Рейтинг наёмников |
| `user_pages/admin.html` | Переходы к административным таблицам |
| `user_pages/member.html` | Задания организации и отчёты на проверку/проверенные |
| `user_pages/mercenary.html` | Текущие и выполненные задания, активные отчёты |
| `sub_user_pages/create_task.html` | Форма задания |
| `sub_user_pages/create_record.html` | Текст отчёта, drag-and-drop изображений, JavaScript → Base64 JSON |
| `sub_user_pages/show_report.html` | Просмотр текста и изображений |
| `admin_tables/CRUD_table_member.html` | Список представителей и действия |
| `admin_tables/CRUD_table_mercenary.html` | Список наёмников и действия |
| `admin_tables/CRUD_table_task.html` | Список заданий и действия |
| `admin_tables/READ_table_member.html` | Просмотр представителя |
| `admin_tables/READ_table_mercenary.html` | Просмотр наёмника |
| `admin_tables/READ_table_task.html` | Просмотр задания |
| `admin_tables/UPDATE_table_member.html` | Изменение представителя |
| `admin_tables/UPDATE_table_mercenary.html` | Изменение наёмника |
| `admin_tables/UPDATE_table_task.html` | Изменение задания |

CSS-файлы оформляют общий интерфейс, страницу администратора, вход, формы создания и CRUD-таблицы соответственно. `static/images/` содержит аватар, пример публикации и иллюстрации с UUID-именами; `static/uploads/` — фотографии `report_<id>_<index>.jpg`. Отдельного JS-модуля и сборщика нет: сценарии встроены в HTML.

## Слои и их ответственность

| Слой | Файл | Ответственность |
|---|---|---|
| HTTP и настройки | app.py | Конфигурация, формы, HTML и редиректы |
| Аутентификация | Flask-Login в app.py | Сессия, вход/выход, загрузка пользователя по ID |
| Права | check_rights | Проверка входа и роли |
| Бизнес-логика | app.py | Регистрация, задания, отчёты, рейтинг, CRUD |
| Данные | models.py | SQLAlchemy, связи и SQLite |
| Представление | templates и static | HTML, CSS, JavaScript, изображения |

Это синхронный монолит без Blueprints, фабрики приложения и отдельного сервисного слоя.

## Модель данных

| Модель / таблица | Поля |
|---|---|
| User / users | `id` PK; уникальный обязательный `username`; обязательный `password`; `registration_date` Date |
| Administrator / administrators | `id_administrator` PK/FK → users.id |
| Member / members | `id_member` PK/FK → users.id; `fraction`, `fraction_code` |
| Mercenary / mercenaries | `id_mercenary` PK/FK → users.id; `telegram_username`; `certificate_code` FK → certificates.certificate_code; `rating_mercenary` Integer; `status_mercenary`, `favourite_fraction`; `price_mercenary` Float |
| Certificate / certificates | `id_certificate` PK; уникальный `certificate_code`; `registration_date_certificate` Date; `place_certification` (по умолчанию «Новый Карфаген»); `giving_certificate` («Единорог») |
| Task / tasks | `id_task` PK; уникальный `id_name_task`; `short_description`, `full_description`, `purpose_task`, `place_task`; `rating_task` Integer; `reward_task` Float; `fraction`; `id_mercenary` FK; `status_task`; даты `start_of_task`, `doing_task`, `end_of_task` |
| Report / reports | `id_report` PK; FK `id_task`, `id_mercenary`; `status_report`; `date_report` Date; `text_report`; `path_image_report` |

ID — Integer, остальные поля без указанного типа — Text. Большинство полей кроме PK и обязательных полей User допускают NULL.

User связан с каждым профилем роли 1:1 через общий PK. Mercenary → Task и Report — 1:N; Task → Report — 1:N. User → профили и Task → отчёты имеют ORM-каскад `all, delete-orphan`. Файлы на диске каскадно не удаляются.

Для Certificate → Mercenary задан `uselist=False`, но `mercenaries.certificate_code` не имеет UNIQUE: единственность проверяет регистрация. Разные профили одного User не исключают друг друга на уровне БД. SQLite foreign_keys pragma явно не включается.

Состояния задания: `Active` → `in_process` → `Done`. Отчёта: `Active` → `Done` или `Rejected`. Наёмник регистрируется как `Alive`; при входе с незавершённым заданием старше семи дней становится `Inactive`; зачёт возвращает `Alive`. Фонового контроля сроков и автоматического удаления заданий нет.

## Как проходит запрос

1. Браузер отправляет GET страницы или POST формы.
2. Flask-Login восстанавливает пользователя из cookie-сессии через load_user.
3. check_rights проверяет роль, если декоратор указан.
4. Обработчик читает форму/параметры, выполняет ORM-запросы и commit.
5. Возвращается HTML или редирект с flash-сообщением. Скачивание возвращает TXT.

При создании отчёта flush получает ID, Base64 сохраняется в файлы, пути относительно static соединяются запятыми в path_image_report, затем выполняется commit. БД и файловая система не объединены одной транзакцией.

## Изоляция данных пользователей

Кабинет наёмника фильтрует записи по ID, кабинет представителя — по организации. Прямые маршруты создания/просмотра/проверки отчётов не проверяют принадлежность объекта. Администратор видит общие таблицы. Подробности — в [known-issues.md](known-issues.md).

## Асинхронность

Обработчики, ORM и запись файлов синхронные. Async-драйверов и фоновой очереди нет. Base64 увеличивает размер запроса и расход памяти.

## Создание таблиц

db.create_all вызывается при импорте в контексте приложения: создаёт отсутствующие таблицы, но не обновляет существующие. Миграций и автоматического наполнения нет. Пути базы — в [configuration.md](configuration.md).
