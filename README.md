Да, это значит одно: сайт пишет не в тот PostgreSQL/не в ту базу, которую ты сейчас смотришь в pgAdmin.

По твоему скрину в pgAdmin есть несколько баз:

ProjectsDB
biha_db
bx
postgres

А наш текущий Db.php работает через:

Bitrix\Main\Application::getConnection()

То есть он пишет в основную базу Битрикса, скорее всего это база:

bx

А ты, похоже, смотришь схему mvc в:

ProjectsDB


---

1. Узнай точно, куда пишет сайт

В PHP-командной строке Битрикса выполни:

use Bitrix\Main\Application;

$conn = Application::getConnection();

$row = $conn->query("
    SELECT
        current_database() AS db_name,
        current_schema() AS schema_name,
        current_user AS db_user,
        inet_server_addr() AS server_ip,
        inet_server_port() AS server_port
")->fetch();

echo '<pre>';
print_r($row);
echo '</pre>';

Он покажет примерно так:

db_name => bx
schema_name => public
db_user => bx_user
server_ip => ...
server_port => 5432

Вот это и есть база, куда сейчас пишет наш MVC.


---

2. Найди таблицу именно там

В pgAdmin открой не ProjectsDB, а ту базу, которую показал current_database().

Например если PHP покажет:

db_name => bx

Тогда смотри:

Servers
  → bx
    → Databases
      → bx
        → Schemas
          → mvc
            → Tables
              → mvc_demo_notes


---

3. Быстрая проверка через PHP

В PHP-командной строке Битрикса выполни:

use Bitrix\Main\Application;

$conn = Application::getConnection();

$rows = $conn->query("
    SELECT *
    FROM mvc.mvc_demo_notes
    ORDER BY id DESC
")->fetchAll();

echo '<pre>';
print_r($rows);
echo '</pre>';

Если сайт показывает заметки, здесь они тоже должны появиться.


---

4. Быстрая проверка в pgAdmin

В той базе, которую покажет current_database(), выполни:

SELECT *
FROM mvc.mvc_demo_notes
ORDER BY id DESC;


---

Почему так получилось

Мы создали таблицу через подключение Битрикса:

Application::getConnection()

А значит таблица создалась в базе Битрикса.

Сейчас логика такая:

MVC → Core\Db → Bitrix Application::getConnection() → база Битрикса

А не так:

MVC → ProjectsDB


---

Что лучше сделать дальше

Пока оставь так. Для обучения Laravel-like MVC это даже хорошо: таблицы demo-проекта лежат рядом с Битриксом и работают стабильно.

А позже мы сделаем как в Laravel:

config/database.php

И сможем выбирать подключение:

Db::connection('bitrix')
Db::connection('projects')
Db::connection('mvc')

Тогда будет понятно и удобно: какие таблицы лежат в базе Битрикса, а какие — в ProjectsDB.