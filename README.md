Да, правильно. pg_master.php не трогаем.
Сделаем готовое решение внутри нашего MVC.

Логика будет такая:

1. MVC пробует использовать старый getPdo().
2. Если getPdo() работает — отлично.
3. Если getPdo() вернул DB_UNAVAILABLE — MVC сам проверяет DB_NODES.
4. MVC ищет мастер через pg_is_in_recovery().
5. Если нашёл мастер — подключается.
6. Если не нашёл — показывает нормальную диагностику.

Меняем только один файл:

/local/mvc/Core/Db.php

Полностью замени его на этот:

<?php

namespace Local\Mvc\Core;

use PDO;
use PDOStatement;
use RuntimeException;
use Throwable;

/**
 * Db
 *
 * Единая точка входа в PostgreSQL для нашего MVC.
 *
 * Важно:
 * pg_master.php НЕ изменяем.
 *
 * Как работает:
 * 1. Подключаем pg_master.php.
 * 2. Пробуем штатный getPdo().
 * 3. Если getPdo() падает с DB_UNAVAILABLE,
 *    наш MVC сам ищет мастер среди DB_NODES.
 */
class Db
{
    private static ?PDO $pdo = null;

    private static ?string $dsnUsed = null;

    private static array $diagnostics = [];

    /**
     * Получить PDO-подключение.
     */
    public static function pdo(): PDO
    {
        if (self::$pdo instanceof PDO) {
            return self::$pdo;
        }

        self::loadPgMaster();

        /**
         * Сначала пробуем использовать существующий getPdo().
         * Это основной путь.
         */
        if (function_exists('\\getPdo')) {
            try {
                $pdo = \getPdo();

                if ($pdo instanceof PDO) {
                    self::configurePdo($pdo);

                    self::$pdo = $pdo;
                    self::$dsnUsed = 'getPdo()';

                    return self::$pdo;
                }

                self::$diagnostics[] = 'getPdo() вернул не PDO';
            } catch (Throwable $e) {
                self::$diagnostics[] = 'getPdo() ERROR=' . $e->getMessage();
            }
        } else {
            self::$diagnostics[] = 'Функция getPdo() не найдена';
        }

        /**
         * Если getPdo() не сработал,
         * включаем резервный механизм MVC.
         */
        $dsn = self::findMasterDsnByMvc();

        $pdo = self::makePdoByMvc($dsn);

        if (!self::isMasterByMvc($pdo)) {
            throw new RuntimeException(
                'MVC_DB_ERROR: найденный узел не является мастером. DSN=' . $dsn
            );
        }

        self::configurePdo($pdo);

        self::$pdo = $pdo;
        self::$dsnUsed = $dsn;

        return self::$pdo;
    }

    /**
     * Подключить старый pg_master.php.
     */
    private static function loadPgMaster(): void
    {
        $pgFile = $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

        if (!is_file($pgFile)) {
            throw new RuntimeException('Файл pg_master.php не найден: ' . $pgFile);
        }

        require_once $pgFile;
    }

    /**
     * Создать PDO напрямую внутри MVC.
     */
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

    /**
     * Проверить, является ли PostgreSQL-узел мастером.
     *
     * pg_is_in_recovery():
     * false / f / 0 — это мастер
     * true  / t / 1 — это реплика
     */
    private static function isMasterByMvc(PDO $pdo): bool
    {
        $inRecovery = $pdo->query('SELECT pg_is_in_recovery()')->fetchColumn();

        return !(
            (string)$inRecovery === 't'
            || (string)$inRecovery === '1'
            || $inRecovery === true
            || $inRecovery === 1
        );
    }

    /**
     * Найти мастер среди DB_NODES.
     */
    private static function findMasterDsnByMvc(): string
    {
        if (!defined('DB_NODES')) {
            throw new RuntimeException('Константа DB_NODES не найдена');
        }

        $nodes = constant('DB_NODES');

        if (!is_array($nodes) || empty($nodes)) {
            throw new RuntimeException('DB_NODES пустой или неверного формата');
        }

        /**
         * Используем отдельный кэш MVC,
         * чтобы не конфликтовать с pg_master.php.
         */
        $cacheKey = 'mvc_db_master_dsn';

        $cached = self::cacheGet($cacheKey);

        if ($cached) {
            try {
                $pdo = self::makePdoByMvc($cached);
                $rec = $pdo->query('SELECT pg_is_in_recovery()')->fetchColumn();

                self::$diagnostics[] = 'MVC CACHED ' . $cached . ' pg_is_in_recovery=' . (string)$rec;

                if (self::isMasterByMvc($pdo)) {
                    return $cached;
                }

                self::cacheSet($cacheKey, '', 1);
            } catch (Throwable $e) {
                self::$diagnostics[] = 'MVC CACHED ' . $cached . ' ERROR=' . $e->getMessage();
                self::cacheSet($cacheKey, '', 1);
            }
        }

        foreach ($nodes as $dsn) {
            try {
                $pdo = self::makePdoByMvc($dsn);

                $rec = $pdo->query('SELECT pg_is_in_recovery()')->fetchColumn();

                self::$diagnostics[] = $dsn . ' pg_is_in_recovery=' . (string)$rec;

                if (self::isMasterByMvc($pdo)) {
                    self::cacheSet($cacheKey, $dsn, 5);

                    return $dsn;
                }
            } catch (Throwable $e) {
                self::$diagnostics[] = $dsn . ' ERROR=' . $e->getMessage();
            }
        }

        throw new RuntimeException(
            'MVC_DB_UNAVAILABLE. DIAG: ' . implode(' | ', self::$diagnostics)
        );
    }

    /**
     * Настроить PDO.
     */
    private static function configurePdo(PDO $pdo): void
    {
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
        $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);
        $pdo->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);
    }

    /**
     * Получить диагностику.
     */
    public static function diagnostics(): array
    {
        return self::$diagnostics;
    }

    /**
     * Узнать, какой DSN был использован.
     */
    public static function dsnUsed(): ?string
    {
        return self::$dsnUsed;
    }

    private static function cacheGet(string $key): ?string
    {
        if (function_exists('apcu_fetch')) {
            $ok = false;
            $val = apcu_fetch($key, $ok);

            return $ok ? (string)$val : null;
        }

        return null;
    }

    private static function cacheSet(string $key, string $value, int $ttl): void
    {
        if (function_exists('apcu_store')) {
            apcu_store($key, $value, $ttl);
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

        if ($row === false) {
            return null;
        }

        return $row;
    }

    /**
     * Получить одно значение.
     */
    public static function value(string $sql, array $params = []): mixed
    {
        $value = self::query($sql, $params)->fetchColumn();

        if ($value === false) {
            return null;
        }

        return $value;
    }

    /**
     * INSERT / UPDATE / DELETE.
     */
    public static function execute(string $sql, array $params = []): int
    {
        return self::query($sql, $params)->rowCount();
    }
}

Что мы сделали

Теперь наш MVC не зависит полностью от того, что getPdo() нашёл мастер.

Он делает так:

Пробую getPdo()
  ↓
если работает — использую его
  ↓
если DB_UNAVAILABLE — сам проверяю DB_NODES
  ↓
ищу узел, где pg_is_in_recovery() = false
  ↓
подключаюсь к нему

Теперь сбрось OPcache

В командной строке PHP Битрикса выполни:

opcache_reset();
echo 'OPcache reset OK';

Проверяй снова

Открой:

https://bitrix24-stage.gaz.ru/local/mvc/db/ping

Если всё хорошо, должен быть ответ вида:

{
  "ok": true,
  "data": {
    "db": {
      "database_name": "ProjectsDB",
      "schema_name": "public",
      "server_time": "..."
    }
  }
}

Если всё равно будет ошибка, теперь она будет подробнее, например:

MVC_DB_UNAVAILABLE. DIAG: ...

И по ней уже будет видно, что именно не так: сеть, пароль, порт, мастер/реплика или драйвер PostgreSQL.