Да, можно сделать на основе этого подключения из sitebuilder. Но сразу важный момент:

sitebuilder-код тоже использует pg_master.php

Вот здесь:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';
$pdo = getPDO();

То есть если PostgreSQL не пускает сервер 192.168.7.109 по pg_hba.conf, то этот код тоже не сможет подключиться. Но мы можем сделать наш Db.php таким же, как в sitebuilder, чтобы не городить лишнюю логику с DB_NODES.

Что было не так у нас

Мы сделали слишком умный Db.php:

пробуем getPdo()
если не получилось — сами перебираем DB_NODES

А в sitebuilder всё проще:

берём getPDO()
ставим search_path
работаем

Сейчас сделаем так же.


---

Замени /local/mvc/Core/Db.php

Полностью замени файл:

/local/mvc/Core/Db.php

на этот:

<?php

namespace Local\Mvc\Core;

use PDO;
use PDOStatement;
use RuntimeException;
use Throwable;

/**
 * Db
 *
 * Единая точка подключения к PostgreSQL для нашего MVC.
 *
 * Сделано по аналогии с sitebuilder:
 * - подключаем pg_master.php
 * - получаем PDO через getPDO/getPdo
 * - настраиваем PDO
 * - выставляем search_path
 */
class Db
{
    private static ?PDO $pdo = null;

    /**
     * Схема по умолчанию.
     *
     * Сейчас ставим sitebuilder, public,
     * потому что в твоём проекте sitebuilder уже использует эту схему.
     */
    private static string $searchPath = 'sitebuilder, public';

    /**
     * Получить PDO.
     */
    public static function pdo(): PDO
    {
        if (self::$pdo instanceof PDO) {
            return self::$pdo;
        }

        $pgFile = $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

        if (!is_file($pgFile)) {
            throw new RuntimeException('PG_MASTER_FILE_NOT_FOUND: ' . $pgFile);
        }

        require_once $pgFile;

        /**
         * Важно:
         * в твоём pg_master.php функция называется getPdo(),
         * а в sitebuilder проверяется getPDO().
         *
         * В PHP имена функций не чувствительны к регистру,
         * но для понятности проверим оба варианта.
         */
        if (function_exists('getPDO')) {
            $pdo = getPDO();
        } elseif (function_exists('getPdo')) {
            $pdo = getPdo();
        } else {
            throw new RuntimeException('FUNCTION_getPDO_NOT_FOUND');
        }

        if (!$pdo instanceof PDO) {
            throw new RuntimeException('getPDO_DID_NOT_RETURN_PDO');
        }

        self::configure($pdo);

        self::$pdo = $pdo;

        return self::$pdo;
    }

    /**
     * Настроить PDO.
     */
    private static function configure(PDO $pdo): void
    {
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
        $pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);

        /**
         * Как в sitebuilder:
         * ставим схему sitebuilder первой.
         *
         * Тогда можно писать:
         * SELECT * FROM sites
         *
         * вместо:
         * SELECT * FROM sitebuilder.sites
         */
        try {
            $pdo->exec('SET search_path TO ' . self::$searchPath);
        } catch (Throwable $e) {
            /**
             * Не валим приложение.
             * Если схема уже задана на уровне подключения,
             * или её нет — это покажет конкретный SQL-запрос позже.
             */
        }
    }

    /**
     * Выполнить SQL-запрос.
     */
    public static function query(string $sql, array $params = []): PDOStatement
    {
        $stmt = self::pdo()->prepare($sql);
        $stmt->execute($params);

        return $stmt;
    }

    /**
     * Получить все строки.
     */
    public static function fetchAll(string $sql, array $params = []): array
    {
        return self::query($sql, $params)->fetchAll();
    }

    /**
     * Получить одну строку.
     */
    public static function fetchOne(string $sql, array $params = []): ?array
    {
        $row = self::query($sql, $params)->fetch();

        return $row !== false ? $row : null;
    }

    /**
     * Получить одно значение.
     */
    public static function value(string $sql, array $params = []): mixed
    {
        $value = self::query($sql, $params)->fetchColumn();

        return $value !== false ? $value : null;
    }

    /**
     * Выполнить INSERT / UPDATE / DELETE.
     */
    public static function execute(string $sql, array $params = []): bool
    {
        $stmt = self::pdo()->prepare($sql);

        return $stmt->execute($params);
    }

    /**
     * Количество затронутых строк.
     */
    public static function affected(string $sql, array $params = []): int
    {
        $stmt = self::pdo()->prepare($sql);
        $stmt->execute($params);

        return $stmt->rowCount();
    }

    /**
     * Последний ID после INSERT.
     */
    public static function lastInsertId(?string $sequence = null): int
    {
        return (int)self::pdo()->lastInsertId($sequence);
    }

    /**
     * Удобный JSON decode, как в sitebuilder.
     */
    public static function jsonDecodeAssoc(mixed $value): array
    {
        if (is_array($value)) {
            return $value;
        }

        if ($value === null || $value === '') {
            return [];
        }

        $decoded = json_decode((string)$value, true);

        return is_array($decoded) ? $decoded : [];
    }
}


---

Что мы этим сделали

Теперь наш MVC подключается к базе почти так же, как sitebuilder:

require pg_master.php
получить PDO через getPDO/getPdo
настроить PDO
SET search_path TO sitebuilder, public

То есть теперь вместо нашей сложной логики будет простая и понятная:

Db::fetchAll(...)
Db::fetchOne(...)
Db::execute(...)
Db::lastInsertId(...)


---

Важный момент

Если после этой замены снова будет:

DB_UNAVAILABLE

значит проблема точно не в MVC.

Значит sitebuilder на этом же сервере тоже не сможет подключиться к PostgreSQL, если он использует этот же pg_master.php.

Потому что ошибка:

нет записи в pg_hba.conf для 192.168.7.109

возникает до SET search_path.

То есть база даже не дала подключиться. До схемы sitebuilder код ещё не дошёл.


---

Проверь снова

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc/db/ping

Если заработает — отлично, дальше сделаем первую модель под таблицы sitebuilder.

Если опять будет DB_UNAVAILABLE, тогда мы продолжим MVC без PostgreSQL или временно подключим модель через Bitrix DB, а доступ к PostgreSQL нужно будет разрешить для сервера 192.168.7.109.