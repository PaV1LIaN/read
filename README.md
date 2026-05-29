Ошибка говорит не про код, а про базу:

relation "mvc.mvc_demo_notes" does not exist

Простыми словами:

Схема mvc есть.
А таблицы mvc_demo_notes внутри неё нет.

Нужно создать таблицу именно здесь:

ProjectsDB → Schemas → mvc → Tables → mvc_demo_notes


---

1. Выполни в pgAdmin этот SQL

CREATE TABLE IF NOT EXISTS mvc.mvc_demo_notes (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);

Потом сразу выполни проверку:

SELECT *
FROM mvc.mvc_demo_notes
ORDER BY id DESC;

Если таблица создана правильно, будет пустая таблица без ошибки.


---

2. На всякий случай выдай права пользователю bx_user

Так как сайт ходит в БД от пользователя bx_user, выполни ещё:

GRANT USAGE ON SCHEMA mvc TO bx_user;

GRANT SELECT, INSERT, UPDATE, DELETE
ON ALL TABLES IN SCHEMA mvc
TO bx_user;

GRANT USAGE, SELECT, UPDATE
ON ALL SEQUENCES IN SCHEMA mvc
TO bx_user;

И для будущих таблиц в этой схеме:

ALTER DEFAULT PRIVILEGES IN SCHEMA mvc
GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO bx_user;

ALTER DEFAULT PRIVILEGES IN SCHEMA mvc
GRANT USAGE, SELECT, UPDATE ON SEQUENCES TO bx_user;


---

3. Проверь, что таблица реально появилась

В pgAdmin раскрой:

Schemas
  → mvc
    → Tables
      → mvc_demo_notes

Если таблицы нет — значит SQL выполнился не в той базе или не выполнился вообще.


---

4. Потом сбрось OPcache

opcache_reset();
echo 'OPcache reset OK';

И снова открой:

/local/mvc_demo/notes

После этого страница должна открыться. Сейчас код уже правильно ищет таблицу:

protected static string $table = 'mvc.mvc_demo_notes';

Проблема только в том, что самой таблицы пока нет в базе.