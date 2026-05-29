Да, правильно. Значит проблема была не в MVC-коде, а в том, что ProjectsDB/pg_hba.conf был настроен не на том PostgreSQL-сервере.

Перед тем как идти дальше, на рабочем сервере проверь 3 вещи.

1. Верни Note на projects

Файл:

/local/mvc_demo/Models/Note.php

Должно быть так:

protected static string $connection = 'projects';

То есть модель заметок теперь должна писать сюда:

ProjectsDB → mvc → mvc_demo_notes


---

2. Проверь подключение с рабочего Битрикс-сервера

В PHP-командной строке Битрикса выполни:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

echo '<pre>';

try {
    $pdo = getPdo();

    echo "PDO OK\n";

    $row = $pdo->query("
        SELECT
            current_database() AS db_name,
            current_schema() AS schema_name,
            current_user AS db_user,
            inet_server_addr() AS server_ip,
            inet_server_port() AS server_port,
            pg_is_in_recovery() AS in_recovery
    ")->fetch(PDO::FETCH_ASSOC);

    print_r($row);
} catch (Throwable $e) {
    echo "ERROR:\n";
    echo $e->getMessage();
}

echo '</pre>';

Нужно увидеть:

PDO OK
db_name => ProjectsDB
db_user => bx_user

И in_recovery должен быть:

f

или пусто/false — то есть это master.


---

3. Проверь таблицу именно в ProjectsDB

В PHP-командной строке Битрикса:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

$pdo = getPdo();

$pdo->exec("SET search_path TO mvc, public");

$row = $pdo->query("
    SELECT to_regclass('mvc.mvc_demo_notes') AS table_name
")->fetch(PDO::FETCH_ASSOC);

echo '<pre>';
print_r($row);
echo '</pre>';

Должно быть:

table_name => mvc.mvc_demo_notes

Если пусто — создай таблицу в ProjectsDB:

CREATE SCHEMA IF NOT EXISTS mvc;

CREATE TABLE IF NOT EXISTS mvc.mvc_demo_notes (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);


---

После этого сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

И проверь:

/local/mvc_demo/notes

Создай новую заметку. Она должна появиться уже в:

ProjectsDB → mvc → mvc_demo_notes

Когда это заработает, двигаемся дальше и сделаем следующий Laravel-like шаг: StoreNoteRequest + UpdateNoteRequest + полноценное редактирование заметки.