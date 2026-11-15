# API

Проект возвращает HTML, редиректы и TXT-файлы. JSON REST API, Swagger, ReDoc и OpenAPI отсутствуют. Все пути ниже взяты из app.py, регистр и завершающие слеши имеют значение. Flask также обслуживает `/static/<path:filename>`; автоматически добавляемые HEAD/OPTIONS в таблицах не перечислены.

## Сводка эндпоинтов

| Метод | Путь | Роль | Назначение |
|---|---|---|---|
| GET | `/` | Любой | Active-задания, рейтинг по убыванию, по два; query page |
| GET | `/rating/` | Любой | Alive-наёмники по рейтингу, по пять; query page |
| GET, POST | `/authentication/` | Любой | Форма входа / вход |
| GET, POST | `/create_member/` | Любой | Форма / регистрация представителя |
| GET, POST | `/create_mercenary/` | Любой | Форма / регистрация наёмника |
| GET, POST | `/goout` | Любой | GET показывает главную; POST завершает сессию |
| GET | `/admin/` | admin | Кабинет администратора |
| GET | `/member/` | member | Задания и отчёты организации |
| GET | `/mercenary/` | mercenary | Собственные задания и активные отчёты |
| GET, POST | `/create_task/` | member, admin | Форма / создание задания |
| GET | `/task_accept/<int:id_task>/` | mercenary | Принятие задания |
| GET | `/task_download/<int:id_task>/` | Любой | Описание задания в TXT |
| GET, POST | `/create_record/<int:id_task>` | mercenary | Форма / создание отчёта |
| GET | `/report_view/<int:id_report>/<string:status_report>` | member, mercenary | Просмотр отчёта |
| GET | `/report_accept/<int:id_report>` | member | Зачёт отчёта и задания, рейтинг |
| GET | `/report_cancel/<int:id_report>` | member | Отклонение отчёта |
| GET | `/CRUD_table_member/` | admin | Список представителей |
| GET | `/CRUD_table_mercenary/` | admin | Список наёмников |
| GET | `/CRUD_table_task/` | admin | Список заданий |
| GET, POST | `/READ_table_member/<int:id>` | admin | Просмотр представителя; POST также только читает |
| GET, POST | `/READ_table_mercenary/<int:id>` | admin | Просмотр наёмника |
| GET, POST | `/READ_table_task/<int:id_task>` | admin | Просмотр задания |
| GET, POST | `/UPDATE_table_member/<int:id>` | admin | Форма / изменение представителя |
| GET, POST | `/UPDATE_table_mercenary/<int:id>` | admin | Форма / изменение наёмника |
| GET, POST | `/UPDATE_table_task/<int:id_task>` | admin | Форма / изменение задания |
| GET, POST | `/delete_member/<int:id>` | admin | Удаление пользователя |
| GET, POST | `/delete_mercenary/<int:id>` | admin | Удаление пользователя |
| GET, POST | `/delete_task/<int:id_task>` | admin | Удаление задания и ORM-каскад отчётов |
| GET, POST | `/object/<int:object_id>/upload_image` | Любой | Незавершённый пример загрузки |
| GET | `/object/<int:object_id>` | Любой | Пример HTML с изображением объекта |

## Авторизация

Flask-Login хранит ID пользователя в подписанной сессии. После POST входа браузер передаёт cookie session. Bearer/JWT не используется. check_rights проверяет только роль; при отсутствии входа или нужной роли перенаправляет на `/` с flash, а не возвращает 401/403.

Обработчик unauthorized отдельно перенаправляет на `/authentication/`, но основные защищённые маршруты используют check_rights. Объектные права не гарантированы — см. [known-issues.md](known-issues.md).

## Пользователи

Формы передаются как application/x-www-form-urlencoded, не JSON.

| POST | Поля | Поведение |
|---|---|---|
| /authentication/ | username, password | Успех → `/`; ошибка → форма входа. Для наёмника проверяется просрочка |
| /create_member/ | username, password, organization_name, organization_code | Создаёт User + Member при верной паре организации/кода и свободном имени |
| /create_mercenary/ | username, password, telegram, cert | Создаёт User + Mercenary при существующем свободном сертификате |
| /goout | Нет | Завершает сессию, перенаправляет на `/` |

При успешной регистрации обычно выполняется редирект на вход. Некоторые неуспешные ветки возвращают форму с кодом 200. Единого JSON-формата ошибок нет. Пароли сохраняются открытым текстом; при неверном входе введённые данные попадают во flash.

## Задачи

POST `/create_task/`: `name_task`, `short_description`, `full_description`, `purpose_task`, `place_task`, `rating_task`, `reward_task`. Для admin дополнительно `fraction`; для member организация берётся из профиля. Создаётся Active-задание без исполнителя, с текущей датой создания. Уникальность имени проверяется обработчиком и ограничением БД.

GET `/task_accept/<id>/` задаёт in_process, текущего исполнителя и дату doing_task. Проверки исходного статуса/занятости нет. Успех перенаправляет на Referer.

GET `/task_download/<id>/` возвращает `text/plain; charset=utf-8`, заголовок Content-Disposition с именем `task_<fraction>_<id>.txt`. Отдельной HTML-страницы подробностей нет: подробности отображаются модальными окнами шаблонов.

## Отчёты

POST `/create_record/<id_task>` принимает `text_report` и `images_data` — JSON-массив строк Base64, например `[]` или `["data:image/jpeg;base64,..."]`. Это поле обычной формы, а не JSON-тело запроса. Невалидный JSON заменяется пустым списком. Отчёт получает Active, текущего автора и дату; файлы сохраняются как report_ID_INDEX.jpg. Редирект — на `/mercenary/`.

В `/report_view/<id>/<status_report>` строка status_report передаётся в шаблон; она не изменяет и не подтверждает статус записи в БД.

Зачёт переводит задание в Done, задаёт дату окончания, пытается обновить отчёты, рейтинг, любимую организацию, цену наёмника и статус Alive. Денежного перевода нет: показывается просьба рассчитаться вручную. Сейчас зачёт содержит ошибки фильтрации и деления на ноль. Отклонение меняет только статус отчёта на Rejected; задание остаётся в работе.

## Административные формы

| POST | Изменяемые поля |
|---|---|
| UPDATE_table_member | username, password, fraction, fraction_code |
| UPDATE_table_mercenary | username, password, telegram_username, rating_mercenary, status_mercenary, favourite_fraction, price_mercenary |
| UPDATE_table_task | id_name_task, short_description, full_description, purpose_task, place_task, rating_task, reward_task, fraction |

После обновления возвращается HTML той же формы. Удаление перенаправляет на Referer. Полноценного отдельного административного создания пользователей нет: используются регистрационные маршруты.

## Коды ответов и незавершённые маршруты

| Код | Ситуация |
|---|---|
| 200 | HTML, TXT или повторный показ формы |
| 302 | Успех операции, отклонение доступа или возврат на форму |
| 400 | В object upload отсутствует image_data; также возможны ошибки разбора запроса |
| 404 / 405 | Неизвестный путь / неподдерживаемый метод |
| 500 | Необработанная ошибка: отсутствующий объект, шаблон, ошибка расчёта и др. |

POST object upload принимает `image_data`, сохраняет object_ID.jpg и перенаправляет на object view. При неудачном декодировании возвращает 500. Привязка к модели оставлена комментарием; GET обращается к отсутствующему upload_image.html. Отчётам этот пример не нужен.

## Примеры на curl

Выполнять локально на учебных данных. Команды сохраняют сессию в отдельном cookie-файле.

```bash
BASE=http://127.0.0.1:5000
curl "$BASE/?page=1"
curl "$BASE/rating/?page=1"

curl -c member.cookies -X POST "$BASE/create_member/" \
  -d 'username=demo-member&password=local-demo&organization_name=Empire&organization_code=empire123'
curl -b member.cookies -c member.cookies -X POST "$BASE/authentication/" \
  -d 'username=demo-member&password=local-demo'
curl -b member.cookies -X POST "$BASE/create_task/" \
  --data-urlencode 'name_task=Учебное задание' \
  --data-urlencode 'short_description=Проверить маршрут' \
  --data-urlencode 'full_description=Описание учебного задания' \
  --data-urlencode 'purpose_task=Разведка' \
  --data-urlencode 'place_task=База' -d 'rating_task=10&reward_task=100'

# Подставить существующий ID задания:
curl "$BASE/task_download/1/" -o task.txt
curl -b member.cookies -X POST "$BASE/goout"
```

Cookie-файлы содержат данные сессии: не добавляйте их в git. Работа с сертификатом и регистрация наёмника описаны в [development.md](development.md).
