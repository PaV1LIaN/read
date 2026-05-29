Ошибка теперь уже другая и понятная:

DB_UNAVAILABLE
/local/php_interface/lib/pg_master.php

Это значит:

Наш MVC теперь пытается идти в ProjectsDB через pg_master.php,
но pg_master.php не может найти доступный master PostgreSQL.

То есть код переключился правильно:

Note → connection projects → pg_master.php → ProjectsDB

Но подключение к ProjectsDB сейчас не работает с этого сервера.


---

Быстро вернуть страницу в рабочее состояние

Пока можно временно вернуть Note на базу Битрикса:

protected static string $connection = 'bitrix';

Но если цель — писать именно в:

ProjectsDB → mvc → mvc_demo_notes

тогда надо починить подключение к ProjectsDB.


---

1. Сначала получим точную диагностику

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

Если снова будет просто:

DB_UNAVAILABLE

значит pg_master.php скрывает детали в error_log.


---

2. Получим диагностику напрямую по нодам

Выполни вот это:

echo '<pre>';

$user = 'bx_user';
$pass = '25##PostPassBX';

$nodes = [
    "pgsql:host=192.168.7.101;port=5432;dbname=ProjectsDB",
    "pgsql:host=192.168.7.102;port=5432;dbname=ProjectsDB",
    "pgsql:host=192.168.7.100;port=5432;dbname=ProjectsDB",
];

foreach ($nodes as $dsn) {
    echo "\n=== {$dsn} ===\n";

    try {
        $pdo = new PDO($dsn, $user, $pass, [
            PDO::ATTR_ERRMODE => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
        ]);

        $row = $pdo->query("
            SELECT
                current_database() AS db_name,
                current_user AS db_user,
                inet_server_addr() AS server_ip,
                inet_server_port() AS server_port,
                pg_is_in_recovery() AS in_recovery
        ")->fetch();

        print_r($row);
    } catch (Throwable $e) {
        echo "ERROR: " . $e->getMessage() . "\n";
    }
}

echo '</pre>';

Вот этот вывод покажет реальную причину: pg_hba, пароль, SSL, master/replica или недоступная нода.


---

3. Скорее всего причина будет pg_hba

Раньше у тебя уже была похожая ошибка:

в pg_hba.conf нет записи для компьютера "192.168.7.109", пользователя "bx_user", базы "ProjectsDB"

Сейчас Битрикс работает с сервера:

192.168.7.110

Значит для PostgreSQL ProjectsDB надо разрешить подключение с 192.168.7.110.

На сервере PostgreSQL в pg_hba.conf должна быть запись примерно такая:

host    ProjectsDB    bx_user    192.168.7.110/32    md5

Или, если пароль SCRAM:

host    ProjectsDB    bx_user    192.168.7.110/32    scram-sha-256

После изменения надо перезагрузить PostgreSQL config:

SELECT pg_reload_conf();


---

4. Пока не починим ProjectsDB, можешь временно вернуть bitrix

В файле:

/local/mvc_demo/Models/Note.php

временно поставь:

protected static string $connection = 'bitrix';

Тогда /notes снова будет работать через базу bx.

А когда pg_master.php начнёт подключаться к ProjectsDB, вернём:

protected static string $connection = 'projects';


---

Сейчас лучше сделай пункт 2 и пришли вывод. По нему я точно скажу, что именно чинить: pg_hba, SSL, пароль, master-ноду или DSN.