1. Общие сведения
1.1. Наименование продукта
LabFlow — учебный веб-сервис учёта сдачи лабораторных работ.

1.2. Цель MVP
Автоматизация и прозрачность процесса сдачи/проверки лабораторных работ для 1 группы (20–30 студентов) и 1 преподавателя по 1 дисциплине.

1.3. Стейкхолдеры
Роль	Интерес	Влияние
Студент	Видеть свои лабы, статусы, дедлайны, комментарии	Высокое
Преподаватель	Управлять лабами, проверять, комментировать	Высокое
Староста	Видеть сводку по группе (read-only)	Среднее
Администратор	Управлять пользователями, ролями	Низкое (в MVP — через админку)
1.4. Scope MVP (что входит)
Аутентификация и RBAC (3 роли).

CRUD лабораторных работ.

Загрузка отчётов студентами.

FSM статусов с аудитом.

Комментарии преподавателя.

Сводка по группе.

Telegram-уведомления (опционально).

Развёртывание через Docker Compose.

1.5. Out-of-scope (что НЕ входит)
SPA, мобильное приложение.

OAuth / SSO.

Экспорт в Excel/CSV.

Мультигрупповость, мультидисциплинарность.

Плагины, публичный API для внешних систем.

1.6. Ограничения
1 группа, 1 дисциплина, 1 преподаватель.

Одновременная нагрузка: до 30 пользователей.

Файлы отчётов: pdf/docx/zip, до 20 МБ.

Хостинг: один сервер, Docker Compose.

1.7. Глоссарий
Термин	Определение
Lab	Лабораторная работа (задание)
Submission	Сдача конкретного студента по конкретной лабе
FSM	Finite State Machine — конечный автомат статусов
RBAC	Role-Based Access Control
is_overdue	Вычисляемый флаг «просрочено», не хранится в БД
StatusHistory	Неизменяемый лог переходов статусов
1.8. Архитектурный стек
Backend: Python 3.11+, Django 5 + DRF (или FastAPI), PostgreSQL 15+.

Frontend: Django Templates + Bootstrap 5 (без SPA на MVP).

Инфраструктура: Docker Compose, Nginx, Gunicorn/Uvicorn.

Интеграции: Telegram Bot API (уведомления).

2. Модель ресурсов (Data Domain)
2.1. ER-диаграмма (Mermaid)
erDiagram
    USER ||--o{ SUBMISSION : "student"
    USER ||--o{ COMMENT : "author"
    USER ||--o{ STATUS_HISTORY : "changed_by"
    LAB ||--o{ SUBMISSION : "has"
    SUBMISSION ||--o{ COMMENT : "has"
    SUBMISSION ||--o{ STATUS_HISTORY : "has"

2.2. Лабораторная работа (Lab)
Поле	Тип	Обяз.	Ограничения	Описание
id	UUID / int	да	PK	Идентификатор
number	int	да	> 0, unique	Порядковый номер
title	varchar(255)	да	3–255 символов	Название
description	text	нет	до 10 000 символов	Задание
deadline	datetime	да	> now при создании	Дедлайн
created_at	datetime	да	auto	Дата создания
Индексы: unique(number), index(deadline).

2.3. Сдача / Отчёт (Submission)
Поле	Тип	Обяз.	Ограничения	Описание
id	UUID / int	да	PK	Идентификатор
lab_id	FK → Lab	да	ON DELETE CASCADE	Лаба
student_id	FK → User	да	ON DELETE CASCADE	Студент
file_path	varchar	нет	pdf/docx/zip, ≤ 20 МБ	Путь к файлу
current_status	enum Status	да	default NOT_STARTED	Текущий статус
submitted_at	datetime	нет	—	Последняя загрузка
created_at	datetime	да	auto	Создание записи
Ограничения: unique(lab_id, student_id), index(current_status).

2.4. Комментарий (Comment)
Поле	Тип	Обяз.	Ограничения	Описание
id	UUID / int	да	PK	Идентификатор
submission_id	FK → Submission	да	ON DELETE CASCADE	Сдача
author_id	FK → User	да	role=Teacher	Автор
text	text	да	1–5000 символов	Текст
created_at	datetime	да	auto	Создан
2.5. Запись истории статусов (StatusHistory)
Поле	Тип	Обяз.	Ограничения	Описание
id	UUID / int	да	PK	Идентификатор
submission_id	FK → Submission	да	ON DELETE CASCADE	Сдача
from_status	enum Status	да	—	Исходный статус
to_status	enum Status	да	—	Новый статус
changed_by	FK → User	да	—	Кто изменил
changed_at	datetime	да	auto	Когда
Правила: только INSERT. UPDATE/DELETE запрещены на уровне сервиса и БД (триггер/права).

2.6. Enum Status
text
NOT_STARTED | IN_PROGRESS | SUBMITTED | NEEDS_REVISION | ACCEPTED
2.7. Миграции и фикстуры
Миграции для всех моделей.

Management-команда seed_demo — создаёт 1 преподавателя, 1 старосту, 20 студентов, 5 лаб, случайные сдачи.

Фикстуры для тестов (отдельная БД test_*).

3. Статусная модель
3.1. Перечень статусов
Код	Название	Цвет UI	Описание
NOT_STARTED	Не начата	серый	Студент ещё не приступал
IN_PROGRESS	В работе	синий	Студент работает
SUBMITTED	Сдана	жёлтый	Файл загружен, ждёт проверки
NEEDS_REVISION	На доработке	оранжевый	Преподаватель вернул
ACCEPTED	Принята	зелёный	Финальный статус
3.2. Правила переходов
Цепочка студента:

#	From	To	Условие	Побочные эффекты	Ошибка при нарушении
S1	NOT_STARTED	IN_PROGRESS	владелец	—	403
S2	IN_PROGRESS	SUBMITTED	файл загружен	submitted_at=now, notify Teacher	400 FILE_REQUIRED
S3	NEEDS_REVISION	IN_PROGRESS	владелец	—	403
S4	*	ACCEPTED	—	—	403 FORBIDDEN_ROLE
S5	SUBMITTED	*	—	—	403 STUDENT_CANNOT_CHANGE_AFTER_SUBMIT
Цепочка преподавателя (только из SUBMITTED):

#	From	To	Условие	Побочные эффекты	Ошибка
T1	SUBMITTED	NEEDS_REVISION	комментарий обязателен	notify Student	400 COMMENT_REQUIRED
T2	SUBMITTED	ACCEPTED	—	notify Student	—
T3	любой другой	—	—	—	403 INVALID_TRANSITION
Общие ограничения:

Откаты назад запрещены (кроме NEEDS_REVISION → IN_PROGRESS).

Студент не может установить ACCEPTED (блок на бэкенде).

Повторная сдача — та же Submission, не новая сущность.

3.3. State-диаграмма
stateDiagram-v2
    [*] --> NOT_STARTED
    NOT_STARTED --> IN_PROGRESS : Student
    IN_PROGRESS --> SUBMITTED : Student (file)
    SUBMITTED --> NEEDS_REVISION : Teacher (comment)
    SUBMITTED --> ACCEPTED : Teacher
    NEEDS_REVISION --> IN_PROGRESS : Student
    ACCEPTED --> [*]

3.4. Sequence-диаграмма: сдача работы
sequenceDiagram
    Student->>API: POST /submissions (file)
    API->>FSM: check(IN_PROGRESS → SUBMITTED)
    FSM-->>API: ok
    API->>DB: update status, submitted_at
    API->>DB: insert StatusHistory
    API->>Telegram: notify Teacher
    API-->>Student: 200 OK

3.5. Вычисляемый флаг «Просрочено» (is_overdue)
Не статус, в БД не хранится.

Формула: now > lab.deadline AND status != ACCEPTED.

Считается in-memory / на уровне queryset (annotate).

Не участвует в FSM.

Тесты: ровно в дедлайн, после, при ACCEPTED.

3.6. Аудит и история
Каждый переход → запись в StatusHistory.

Поля: from_status, to_status, changed_by, changed_at.

Запись immutable.

Все изменения — в одной транзакции с обновлением Submission.

4. Ролевая модель и разграничение доступа
4.1. Роли
4.1.1. Студент (Student)
Доступ: только свои Submission.

Права:

Просмотр списка своих лаб и статусов.

Загрузка файла отчёта.

Переходы: NOT_STARTED → IN_PROGRESS, IN_PROGRESS → SUBMITTED, NEEDS_REVISION → IN_PROGRESS.

Запреты: CRUD лаб, комментарии, чужие работы, сводка.

4.1.2. Преподаватель (Teacher)
Доступ: все работы группы.

Права:

Полный CRUD Lab.

Просмотр всех Submission.

Переходы только из SUBMITTED: → NEEDS_REVISION / ACCEPTED.

Комментарии.

Сводка и статистика.

4.1.3. Староста (Headman)
Доступ: read-only.

Права: просмотр лаб, сдач группы, сводки.

Запреты: смена статусов, загрузка, комментарии, CRUD лаб.

4.2. Матрица доступа (RBAC)
Операция / Ресурс	Студент	Преподаватель	Староста
GET /labs	✅ (свои статусы)	✅	✅
POST/PUT/PATCH/DELETE /labs	❌	✅	❌
POST /submissions (upload)	✅ (свой)	❌	❌
GET /submissions/{id}	✅ (свой)	✅	✅
PATCH /submissions/{id}/status	✅ (по цепочке)	✅ (из SUBMITTED)	❌
POST /submissions/{id}/comments	❌	✅	❌
GET /submissions/{id}/comments	✅ (свой)	✅	✅
GET /summary	❌	✅	✅
4.3. Object-level правила
Студент: submission.student_id == request.user.id, иначе 404 (не 403 — чтобы не раскрывать существование).

Преподаватель/староста: доступ ко всем Submission группы.

Комментарии: студент видит только комментарии к своей сдаче.

4.4. Уровни проверки
Аутентификация — middleware.

Роль — permission-класс (IsTeacher, IsStudent, IsHeadman).

Object-level — в сервисе/queryset.

FSM — сервис переходов.

5. API Эндпоинты
Общие правила:

Base URL: /api/v1/.

Формат: JSON (кроме загрузки файлов — multipart/form-data).

Аутентификация: сессия или JWT (Authorization: Bearer <token>).

Ошибки: единый формат { "error": "CODE", "detail": "..." }.

5.1. Авторизация
Метод	URL	Роль	Описание
POST	/auth/login	все	Вход, выдача токена/сессии
POST	/auth/logout	все	Выход
GET	/auth/me	все	Текущий пользователь
Пример запроса:

http
POST /api/v1/auth/login
Content-Type: application/json

{ "email": "student@example.com", "password": "secret" }
Ответ 200:

json
{ "token": "eyJ...", "user": { "id": 1, "role": "Student", "full_name": "Иванов И." } }
Ошибки: 400 INVALID_CREDENTIALS, 429 TOO_MANY_ATTEMPTS.

5.2. Лабораторные работы (Labs)
Метод	URL	Роль	Описание
GET	/labs	все	Список
POST	/labs	Teacher	Создание
GET	/labs/{id}	все	Детали
PUT/PATCH	/labs/{id}	Teacher	Редактирование
DELETE	/labs/{id}	Teacher	Удаление (optional)
GET /labs (для студента):

json
[
  { "id": 1, "number": 1, "title": "Лаба 1", "deadline": "2025-10-01T23:59:00Z",
    "my_status": "IN_PROGRESS", "is_overdue": false }
]
POST /labs:

json
{ "number": 2, "title": "Лаба 2", "description": "...", "deadline": "2025-11-01T23:59:00Z" }
Ошибки: 400 DEADLINE_IN_PAST, 409 NUMBER_ALREADY_EXISTS, 403 FORBIDDEN_ROLE.

5.3. Сдачи и Отчёты (Submissions)
Метод	URL	Роль	Описание
GET	/submissions/my	Student	Мои сдачи
GET	/labs/{lab_id}/submissions	Teacher, Headman	Сдачи по лабе
POST	/submissions	Student	Загрузка файла
GET	/submissions/{id}	по правам	Детали
PATCH	/submissions/{id}/status	Student/Teacher	Смена статуса
GET	/submissions/{id}/history	по правам	История
POST /submissions (multipart/form-data):

lab_id, file (pdf/docx/zip, ≤ 20 МБ).

Автосоздание записи со статусом IN_PROGRESS → SUBMITTED.

Повторная загрузка обновляет submitted_at.

PATCH /submissions/{id}/status:

json
{ "to_status": "ACCEPTED", "comment": "Зачтено" }
Ошибки: 400 INVALID_TRANSITION, 400 COMMENT_REQUIRED, 403 FORBIDDEN_ROLE, 413 FILE_TOO_LARGE.

5.4. Комментарии (Comments)
Метод	URL	Роль	Описание
POST	/submissions/{id}/comments	Teacher	Добавить
GET	/submissions/{id}/comments	Teacher, Headman, владелец	Список
POST:

json
{ "text": "Добавь раздел с выводами" }
Валидация: непустой, ≤ 5000 символов.

5.5. Сводка (Summary)
Метод	URL	Роль	Описание
GET	/summary	Teacher, Headman	Агрегаты по группе
Фильтры: ?lab_id=, ?student_id=.
Ответ:

json
{
  "total": 25, "submitted": 8, "accepted": 10,
  "needs_revision": 4, "overdue": 3
}
5.6. Коды ошибок (сводно)
Код	HTTP	Когда
INVALID_CREDENTIALS	401	Неверный логин/пароль
FORBIDDEN_ROLE	403	Роль не имеет права
INVALID_TRANSITION	400	Запрещённый переход FSM
COMMENT_REQUIRED	400	NEEDS_REVISION без комментария
FILE_REQUIRED	400	SUBMITTED без файла
FILE_TOO_LARGE	413	> 20 МБ
NOT_FOUND	404	Объект не найден/недоступен
6. Требования к реализации и DoD
6.1. Функциональные требования
RBAC на каждом эндпоинте (permission-классы).

FSM в отдельном сервисном слое.

Все изменения — в PostgreSQL.

Файлы — в MEDIA_ROOT, путь в БД.

Telegram-уведомления при смене статуса.

6.2. Нефункциональные требования (НФТ)
Категория	Требование
Производительность	p95 < 300 мс при 30 одновременных пользователях
Безопасность	HTTPS, CSRF, XSS, rate-limit на login (5/мин)
Логирование	structlog, JSON, уровень INFO+ в prod
Резервное копирование	ежедневный pg_dump
Доступность	99% в учебный период
Размер файла	≤ 20 МБ
Совместимость	Chrome, Firefox, Safari (последние 2 версии)
6.3. Критерии готовности (DoD)
□ docker-compose up поднимает всё окружение без ошибок.
□ Все Unit/Integration тесты проходят.
□ Покрытие ≥ 80%.
□ RBAC проверяется на бэкенде для каждого эндпоинта.
□ FSM блокирует некорректные переходы.
□ is_overdue корректно считается в списках и сводке.
□ Frontend отображает корректные кнопки для каждой роли.
□ Telegram-уведомления приходят (если в скоупе).
□ Swagger-документация актуальна.
□ README с инструкцией по запуску.
6.4. Трассируемость (пример)
Требование	Задача	Тест
FSM: студент не ставит ACCEPTED	B4.3	TC-01
is_overdue в сводке	B5.3, B9.1	TC-10
RBAC: студент не видит чужое	B3.5, B7.7	TC-20
7. UI/UX (детализация)
7.1. Карта экранов по ролям
Студент: login → my-labs → lab/{id} → upload → history/comments.
Преподаватель: login → labs (CRUD) → lab/{id}/submissions → submission/{id} → summary.
Староста: login → labs (read) → lab/{id}/submissions (read) → summary (read).

7.2. Состояния экранов
loading — спиннер.

empty — «Нет лабораторных / Нет сдач».

error — alert + retry.

success — flash-сообщение.

7.3. Цвета статусов
NOT_STARTED — серый, IN_PROGRESS — синий, SUBMITTED — жёлтый, NEEDS_REVISION — оранжевый, ACCEPTED — зелёный.

8. Инфраструктура
8.1. Переменные окружения
Переменная	Назначение	Пример
DB_HOST	хост PostgreSQL	db
DB_NAME	имя БД	labflow
DB_USER	пользователь	labflow
DB_PASSWORD	пароль	<secret>
SECRET_KEY	ключ Django	<random>
DEBUG	режим	False
TELEGRAM_TOKEN	токен бота	123:ABC
MEDIA_ROOT	путь к файлам	/app/media
8.2. Схема деплоя

flowchart LR
    Client --> Nginx
    Nginx -->|/api| Backend
    Nginx -->|/static,/media| Static
    Backend --> Postgres
    Backend --> Telegram

8.3. Docker Compose
Сервисы: db, backend, nginx, bot (opt).

Тома: pgdata, media.

Health-check для db и backend.

9. Тестирование
9.1. Пирамида
Unit: FSM, is_overdue, permissions.

Integration: полный цикл сдачи.

E2E: Playwright (2 сценария).

9.2. Матрица тест-кейсов (фрагмент)
ID	Сценарий	Роль	Ожидание
TC-01	Студент ставит ACCEPTED	Student	403
TC-02	Teacher принимает SUBMITTED	Teacher	200 + StatusHistory
TC-03	Teacher принимает IN_PROGRESS	Teacher	403 INVALID_TRANSITION
TC-04	NEEDS_REVISION без комментария	Teacher	400 COMMENT_REQUIRED
TC-10	is_overdue после дедлайна	—	true
TC-11	is_overdue при ACCEPTED	—	false
TC-20	Студент читает чужое	Student	404
9.3. Нагрузочное
k6/Locust: 30 одновременных пользователей, p95 < 300 мс.

10. Приоритизация (MoSCoW)
Приоритет	Задачи
Must	B1–B9, F1–F6, Q1–Q3, Q5
Should	B10 (Telegram), B11 (Admin), F7, Q6
Could	B6.5 (DELETE labs), F2.3 (drag&drop), F7.4 (сортировка)
Won't (MVP)	SPA, OAuth, мобильное приложение, экспорт в Excel
