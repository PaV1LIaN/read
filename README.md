Да, значит всё подтвердилось:

sitebuilder-подключение = тот же pg_master.php
pg_master.php = DB_UNAVAILABLE

То есть через PostgreSQL сейчас не пойдём, пока 192.168.7.109 не разрешат в pg_hba.conf.

Но для нашего MVC это не стопор. Правильнее сделать так:

Db.php      — база самого Битрикса
PgDb.php    — PostgreSQL потом, когда доступ появится

Сейчас сделаем рабочий Db.php через стандартное подключение Битрикса.


---

1. Замени /local/mvc/Core/Db.php

<?php

namespace Local\Mvc\Core;

use Bitrix\Main\Application;
use Bitrix\Main\DB\Connection;
use RuntimeException;

/**
 * Db
 *
 * База данных Битрикса.
 *
 * Важно:
 * Это НЕ ProjectsDB PostgreSQL.
 * Это стандартная база, к которой уже подключён сам Битрикс.
 *
 * Поэтому она должна работать всегда, если работает портал.
 */
class Db
{
    private static ?Connection $connection = null;

    public static function connection(): Connection
    {
        if (self::$connection instanceof Connection) {
            return self::$connection;
        }

        self::$connection = Application::getConnection();

        return self::$connection;
    }

    public static function type(): string
    {
        return self::connection()->getType();
    }

    public static function fetchAll(string $sql, array $params = []): array
    {
        $sql = self::prepareSql($sql, $params);

        $result = self::connection()->query($sql);

        $rows = [];

        while ($row = $result->fetch()) {
            $rows[] = $row;
        }

        return $rows;
    }

    public static function fetchOne(string $sql, array $params = []): ?array
    {
        $sql = self::prepareSql($sql, $params);

        $result = self::connection()->query($sql);
        $row = $result->fetch();

        return $row !== false ? $row : null;
    }

    public static function value(string $sql, array $params = []): mixed
    {
        $row = self::fetchOne($sql, $params);

        if (!$row) {
            return null;
        }

        $values = array_values($row);

        return $values[0] ?? null;
    }

    public static function execute(string $sql, array $params = []): bool
    {
        $sql = self::prepareSql($sql, $params);

        self::connection()->queryExecute($sql);

        return true;
    }

    public static function lastInsertId(): int
    {
        if (method_exists(self::connection(), 'getInsertedId')) {
            return (int)self::connection()->getInsertedId();
        }

        return 0;
    }

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

    /**
     * Простая подстановка параметров.
     *
     * Например:
     * SELECT * FROM b_user WHERE ID = :id
     *
     * Db::fetchOne($sql, ['id' => 5])
     */
    private static function prepareSql(string $sql, array $params = []): string
    {
        if (empty($params)) {
            return $sql;
        }

        uksort($params, static function ($a, $b) {
            return strlen((string)$b) <=> strlen((string)$a);
        });

        foreach ($params as $key => $value) {
            $placeholder = ':' . ltrim((string)$key, ':');
            $sql = str_replace($placeholder, self::quote($value), $sql);
        }

        return $sql;
    }

    private static function quote(mixed $value): string
    {
        if ($value === null) {
            return 'NULL';
        }

        if (is_bool($value)) {
            return $value ? '1' : '0';
        }

        if (is_int($value) || is_float($value)) {
            return (string)$value;
        }

        if (is_array($value)) {
            throw new RuntimeException('Db::quote не поддерживает массивы');
        }

        return "'" . self::connection()->getSqlHelper()->forSql((string)$value) . "'";
    }
}


---

2. Замени /local/mvc/Controllers/DbController.php

<?php

namespace Local\Mvc\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Db;
use Local\Mvc\Core\Response;
use Throwable;

/**
 * DbController
 *
 * Проверка подключения к базе Битрикса.
 */
class DbController extends Controller
{
    public function test(): Response
    {
        try {
            $row = Db::fetchOne("
                SELECT 
                    1 AS test_value,
                    NOW() AS server_time
            ");

            return $this->render('db/test', [
                'title' => 'Проверка базы Битрикса',
                'row' => $row,
                'dbType' => Db::type(),
                'error' => null,
            ]);
        } catch (Throwable $e) {
            return $this->render('db/test', [
                'title' => 'Ошибка базы Битрикса',
                'row' => null,
                'dbType' => null,
                'error' => $e->getMessage(),
            ]);
        }
    }

    public function ping(): Response
    {
        try {
            $row = Db::fetchOne("
                SELECT 
                    1 AS test_value,
                    NOW() AS server_time
            ");

            return $this->success([
                'db' => [
                    'type' => Db::type(),
                    'test_value' => $row['test_value'] ?? null,
                    'server_time' => $row['server_time'] ?? null,
                ],
            ]);
        } catch (Throwable $e) {
            return $this->error('BITRIX_DB_ERROR', [
                'message' => $e->getMessage(),
            ], 500);
        }
    }

    public function users(): Response
    {
        try {
            $users = Db::fetchAll("
                SELECT 
                    ID,
                    LOGIN,
                    NAME,
                    LAST_NAME,
                    EMAIL
                FROM b_user
                ORDER BY ID DESC
                LIMIT 10
            ");

            return $this->success([
                'users' => $users,
            ]);
        } catch (Throwable $e) {
            return $this->error('USERS_QUERY_ERROR', [
                'message' => $e->getMessage(),
            ], 500);
        }
    }
}


---

3. Замени /local/mvc/Views/db/test.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Проверка базы') ?>
    </h1>

    <?php if (!empty($error)): ?>
        <p class="mvc-page-text" style="color: #b91c1c;">
            Ошибка:
        </p>

        <div class="mvc-info">
            <pre><?= htmlspecialcharsbx($error) ?></pre>
        </div>
    <?php else: ?>
        <p class="mvc-page-text">
            Подключение к базе Битрикса работает.
        </p>

        <div class="mvc-info">
            <b>Информация:</b>

            <ol>
                <li>
                    Тип подключения:
                    <span class="mvc-code">
                        <?= htmlspecialcharsbx($dbType ?? '') ?>
                    </span>
                </li>

                <li>
                    Тестовое значение:
                    <span class="mvc-code">
                        <?= htmlspecialcharsbx($row['test_value'] ?? '') ?>
                    </span>
                </li>

                <li>
                    Время сервера БД:
                    <span class="mvc-code">
                        <?= htmlspecialcharsbx($row['server_time'] ?? '') ?>
                    </span>
                </li>
            </ol>
        </div>
    <?php endif; ?>
</div>


---

4. Обнови /local/mvc/routes.php

Добавь маршрут /db/users.

<?php

use Local\Mvc\Core\Router;
use Local\Mvc\Controllers\HomeController;
use Local\Mvc\Controllers\DbController;

/** @var Router $router */

$router->get('/', [HomeController::class, 'index']);

$router->get('/ping', [HomeController::class, 'ping']);

$router->get('/go-home', [HomeController::class, 'goHome']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/db/test', [DbController::class, 'test']);

$router->get('/db/ping', [DbController::class, 'ping']);

$router->get('/db/users', [DbController::class, 'users']);


---

5. Проверь

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc/db/ping

Должно быть примерно:

{
  "ok": true,
  "data": {
    "db": {
      "type": "mysql",
      "test_value": "1",
      "server_time": "2026-..."
    }
  }
}

Потом проверь пользователей:

https://bitrix24-stage.gaz.ru/local/mvc/db/users

Если вернёт список пользователей — значит наш MVC теперь умеет работать с базой Битрикса.


---

Что мы сделали простыми словами

Было:

MVC пытался подключиться к внешней PostgreSQL ProjectsDB
PostgreSQL не пускал сервер 192.168.7.109

Стало:

MVC использует базу самого Битрикса
Битрикс уже к ней подключён
значит MVC может работать

А PostgreSQL мы потом вынесем отдельно:

Db.php    — база Битрикса
PgDb.php  — ProjectsDB PostgreSQL

Так фреймворк не будет зависеть от одной проблемной внешней базы.