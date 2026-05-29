Понял. Тут главное: сайту нельзя верить “на глаз” по pgAdmin-дереву. Нужно найти таблицу SQL-запросом.

Если сайт показывает заметки, значит таблица точно где-то есть, потому что Note::latest() делает:

SELECT *
FROM mvc.mvc_demo_notes
ORDER BY id DESC
LIMIT 20

И если страница открывается без ошибки, этот запрос выполняется.


---

1. Выполни в PHP-командной строке Битрикса

use Bitrix\Main\Application;

$conn = Application::getConnection();

echo '<pre>';

echo "=== CONNECTION ===\n";
print_r($conn->query("
    SELECT
        current_database() AS db_name,
        current_schema() AS current_schema,
        current_user AS db_user,
        current_setting('search_path') AS search_path,
        inet_server_addr() AS server_ip,
        inet_server_port() AS server_port
")->fetch());

echo "\n=== FIND MVC TABLES ===\n";
print_r($conn->query("
    SELECT
        table_schema,
        table_name
    FROM information_schema.tables
    WHERE table_name ILIKE '%mvc%'
       OR table_schema ILIKE '%mvc%'
    ORDER BY table_schema, table_name
")->fetchAll());

echo "\n=== FIND NOTES TABLES ===\n";
print_r($conn->query("
    SELECT
        table_schema,
        table_name
    FROM information_schema.tables
    WHERE table_name ILIKE '%note%'
    ORDER BY table_schema, table_name
")->fetchAll());

echo "\n=== TO_REGCLASS ===\n";
print_r($conn->query("
    SELECT
        to_regclass('mvc.mvc_demo_notes') AS mvc_table,
        to_regclass('public.mvc_demo_notes') AS public_table
")->fetch());

echo "\n=== DATA CHECK ===\n";
print_r($conn->query("
    SELECT *
    FROM mvc.mvc_demo_notes
    ORDER BY id DESC
")->fetchAll());

echo '</pre>';

Вот этот вывод самый важный.


---

2. Что мы по нему поймём

Если будет:

db_name => bx
current_schema => public
search_path => "$user", public

это не значит, что таблица лежит в public.

current_schema = public означает только:

если написать SELECT * FROM table_name без схемы,
PostgreSQL будет искать в public

Но наша модель пишет явно:

mvc.mvc_demo_notes

Значит таблица должна быть именно в схеме mvc.


---

3. Если в выводе будет mvc.mvc_demo_notes

Например:

table_schema => mvc
table_name => mvc_demo_notes

но в pgAdmin ты её не видишь, значит одно из двух:

1. pgAdmin открыт не на том сервере / не на той базе
2. дерево pgAdmin не обновилось

В pgAdmin сделай:

Schemas → правой кнопкой → Refresh
mvc → Tables → правой кнопкой → Refresh

И обязательно проверь, что открыт именно тот сервер/IP и база, которые покажет PHP:

server_ip
server_port
db_name


---

4. Если в выводе будет public.mvc_demo_notes

Тогда у тебя модель сейчас почему-то смотрит в public, а не в mvc.

Но по прошлой ошибке было именно:

SELECT * FROM mvc.mvc_demo_notes

так что скорее всего таблица всё-таки в mvc, просто ты смотришь не тот узел в pgAdmin.


---

5. Если PHP покажет данные, но pgAdmin нигде не показывает

Тогда почти наверняка у тебя в pgAdmin открыт другой сервер.

У тебя может быть так:

pgAdmin показывает ProjectsDB на одном узле
Битрикс пишет в bx на другом узле
или в primary, а pgAdmin смотрит replica/другой host

Сравни именно:

server_ip
server_port
db_name

из PHP с тем, что у тебя в pgAdmin.


---

Пришли сюда вывод блоков:

CONNECTION
FIND MVC TABLES
FIND NOTES TABLES
TO_REGCLASS

и я точно скажу, где лежит таблица.