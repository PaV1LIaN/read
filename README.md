Это уже не ошибка MVC. Теперь мы узнали настоящую причину:

в pg_hba.conf нет записи для компьютера "192.168.7.109", пользователя "bx_user", базы "ProjectsDB", без шифрования

Простыми словами:

Сервер Битрикса: 192.168.7.109
пытается подключиться к PostgreSQL
как пользователь bx_user
к базе ProjectsDB

PostgreSQL отвечает:
"Я такого клиента без SSL не пускаю"

Что сделаем в MVC

Сначала попробуем не трогать PostgreSQL, а заставим наш MVC подключаться к базе через SSL.

В файле:

/local/mvc/Core/Db.php

найди функцию:

private static function makePdoByMvc(string $dsn): PDO

и полностью замени её на эту:

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
     * Важно.
     *
     * PostgreSQL сказал:
     * "нет записи pg_hba.conf ... без шифрования"
     *
     * Значит, возможно, база ждёт SSL-подключение.
     * Поэтому добавляем sslmode=require, если его ещё нет.
     */
    if (stripos($dsn, 'sslmode=') === false) {
        $dsn .= ';sslmode=require';
    }

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

Потом в PHP-командной строке Битрикса выполни:

opcache_reset();
echo 'OPcache reset OK';

И снова открой:

https://bitrix24-stage.gaz.ru/local/mvc/db/ping

Что должно измениться

Сейчас ошибка была:

без шифрования

После sslmode=require есть шанс, что подключение заработает.

Если заработает — отлично, идём дальше.

Если ошибка станет другой, например:

нет записи pg_hba.conf ... с шифрованием

тогда это значит: MVC уже сделал всё, что мог, и PostgreSQL всё равно не разрешает подключение серверу 192.168.7.109.

Тогда на PostgreSQL-серверах надо добавить доступ в pg_hba.conf.

Если всё-таки понадобится правка PostgreSQL

На каждом PostgreSQL-узле:

192.168.7.100
192.168.7.101
192.168.7.102

в pg_hba.conf нужна строка примерно такая:

hostssl    ProjectsDB    bx_user    192.168.7.109/32    scram-sha-256

Если у вас используется md5, тогда:

hostssl    ProjectsDB    bx_user    192.168.7.109/32    md5

После этого перезагрузить конфиг PostgreSQL:

sudo systemctl reload postgresql

или через SQL:

SELECT pg_reload_conf();

Главное

Наш MVC сейчас работает правильно. Он дошёл до PostgreSQL и получил честный ответ:

PostgreSQL не разрешает подключение серверу 192.168.7.109

Сначала пробуем решение внутри MVC через sslmode=require. Если не поможет — это уже настройка доступа PostgreSQL в pg_hba.conf.