Отлично, теперь диагноз окончательный:

server does not support SSL, but SSL was required

Значит моя проба с sslmode=require не подходит. PostgreSQL-серверы не поддерживают SSL.

Итог:

SSL включить из MVC нельзя.
pg_hba.conf обойти из PHP нельзя.

Это как дверь в кабинет: наш код подошёл к двери, но охрана PostgreSQL сказала:

Сервер 192.168.7.109 не разрешён для bx_user к базе ProjectsDB.

Что делаем сейчас

Сначала откатываем SSL в нашем MVC.

В файле:

/local/mvc/Core/Db.php

найди функцию:

private static function makePdoByMvc(string $dsn): PDO

и замени её обратно на такую:

private static function makePdoByMvc(string $dsn): PDO
{
    if (!defined('DB_USER')) {
        throw new RuntimeException('Константа DB_USER не найдена');
    }

    if (!defined('DB_PASS')) {
        throw new RuntimeException('Константа DB_PASS не найдена');
    }

    if (stripos($dsn, 'pgsql:') !== 0) {
        $dsn = 'pgsql:' . $dsn;
    }

    /**
     * ВАЖНО:
     * sslmode=require убрали, потому что PostgreSQL ответил:
     * server does not support SSL
     */
    return new PDO(
        $dsn,
        constant('DB_USER'),
        constant('DB_PASS'),
        [
            PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
            PDO::ATTR_DEFAULT_FETCH_MODE => PDO::FETCH_ASSOC,
            PDO::ATTR_EMULATE_PREPARES   => false,
        ]
    );
}

Потом сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Почему MVC не может это исправить кодом

Ошибка была такая:

в pg_hba.conf нет записи для компьютера "192.168.7.109", пользователя "bx_user", базы "ProjectsDB", без шифрования

Это значит, что на PostgreSQL-серверах нет разрешения для подключения:

кто подключается: 192.168.7.109
пользователь: bx_user
база: ProjectsDB
тип подключения: без SSL

Чтобы база пустила наш MVC, на PostgreSQL-серверах нужно добавить разрешение в pg_hba.conf.

На каждом узле PostgreSQL:

192.168.7.100
192.168.7.101
192.168.7.102

нужна строка примерно такая:

host    ProjectsDB    bx_user    192.168.7.109/32    md5

или, если используется scram-sha-256:

host    ProjectsDB    bx_user    192.168.7.109/32    scram-sha-256

После этого на PostgreSQL:

sudo systemctl reload postgresql

или через SQL:

SELECT pg_reload_conf();

Что по нашему MVC

Наш MVC сейчас работает правильно. Он дошёл до базы и показал реальную проблему.

Цепочка такая:

/local/mvc/db/ping
  ↓
DbController
  ↓
Db
  ↓
getPdo()
  ↓
DB_UNAVAILABLE
  ↓
MVC сам проверил DB_NODES
  ↓
PostgreSQL ответил: нет доступа для 192.168.7.109

То есть код MVC чинить дальше не нужно.
Нужно либо дать доступ 192.168.7.109 в PostgreSQL, либо для обучения временно продолжить без PostgreSQL и сделать следующий шаг на тестовой модели без базы.

Я бы сейчас сделал так: не останавливаемся на PostgreSQL, а продолжаем строить MVC. Следующим шагом можно сделать Auth и Access через пользователя Битрикса, а подключение к PostgreSQL вернём, когда для 192.168.7.109 добавят доступ в pg_hba.conf.