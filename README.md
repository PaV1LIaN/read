Такая же ошибка значит таблица создана не в той базе, куда сейчас ходит Битрикс.

По trace видно важное:

/local/mvc/Core/Db.php
→ Bitrix\Main\DB\PgsqlConnection

То есть наш Db.php сейчас работает через основное подключение Битрикса, а не через тот ProjectsDB, который ты видишь в pgAdmin.

Нужно создать таблицу именно в той БД, которую использует Битрикс.


---

1. Сначала узнаем, куда реально подключён Битрикс

В админке Битрикса открой PHP-командную строку и выполни:

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

$row2 = $conn->query("
    SELECT to_regclass('mvc.mvc_demo_notes') AS table_name
")->fetch();

echo '<pre>';
print_r($row2);
echo '</pre>';

Если увидишь:

table_name =>

пустое значение, значит для Битрикса таблицы реально нет.


---

2. Создадим таблицу прямо через подключение Битрикса

В той же PHP-командной строке выполни:

use Bitrix\Main\Application;

$conn = Application::getConnection();

$conn->queryExecute("
    CREATE SCHEMA IF NOT EXISTS mvc
");

$conn->queryExecute("
    CREATE TABLE IF NOT EXISTS mvc.mvc_demo_notes (
        id BIGSERIAL PRIMARY KEY,
        title VARCHAR(255) NOT NULL,
        body TEXT NULL,
        created_at TIMESTAMP NULL,
        updated_at TIMESTAMP NULL
    )
");

$row = $conn->query("
    SELECT to_regclass('mvc.mvc_demo_notes') AS table_name
")->fetch();

echo '<pre>';
print_r($row);
echo '</pre>';

Должно вывести:

table_name => mvc.mvc_demo_notes


---

3. Проверь страницу

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/notes


---

Почему так произошло

Ты создал схему mvc в pgAdmin, скорее всего здесь:

ProjectsDB → Schemas → mvc

А Битрикс, возможно, подключён к другой базе, например:

bx

Поэтому pgAdmin показывает схему, но сайт её не видит.

Главное правило:

Таблицы для нашего MVC должны быть созданы в той базе,
которую показывает current_database() из подключения Bitrix\Main\Application::getConnection().

Сделай пункт 1 и пришли результат db_name, если после создания через PHP-командную строку ошибка останется.