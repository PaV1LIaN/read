Идём дальше. Сейчас сделаем слой базы данных.

У нас уже есть:

Request    — что пришло от пользователя
Router     — куда отправить запрос
Controller — что выполнить
View       — что показать
Layout     — общая оболочка страницы
Response   — что вернуть пользователю

Теперь добавим:

Db    — подключение к PostgreSQL
Model — удобная работа с таблицами

Зачем нужен Db

Без Db в каждом контроллере пришлось бы писать:

require_once $_SERVER['DOCUMENT_ROOT'].'/local/php_interface/lib/pg_master.php';

$pdo = getPdo();

$stmt = $pdo->prepare('SELECT * FROM table');
$stmt->execute();
$rows = $stmt->fetchAll();

И это будет повторяться везде.

А мы хотим так:

$rows = Db::fetchAll('SELECT * FROM sitebuilder.sites');

То есть Db — это как общий переходник к базе.


---

Шаг 1. Создаём /local/mvc/Core/Db.php

Создай файл:

/local/mvc/Core/Db.php

Код:

<?php

namespace Local\Mvc\Core;

use PDO;
use PDOStatement;
use RuntimeException;

/**
 * Db
 *
 * Это единая точка входа в PostgreSQL.
 *
 * Простыми словами:
 * если кому-то нужна база — он идёт сюда.
 */
class Db
{
    /**
     * Здесь будем хранить одно PDO-подключение.
     *
     * Чтобы не подключаться к базе 10 раз за один запрос.
     */
    private static ?PDO $pdo = null;

    /**
     * Получить PDO.
     */
    public static function pdo(): PDO
    {
        if (self::$pdo instanceof PDO) {
            return self::$pdo;
        }

        /**
         * Подключаем твой существующий файл с PostgreSQL.
         *
         * У тебя он уже используется в других проектах:
         * /local/php_interface/lib/pg_master.php
         */
        $pgFile = $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

        if (!is_file($pgFile)) {
            throw new RuntimeException('Файл подключения к PostgreSQL не найден: ' . $pgFile);
        }

        require_once $pgFile;

        /**
         * Ожидаем, что в pg_master.php есть функция getPdo().
         */
        if (!function_exists('getPdo')) {
            throw new RuntimeException('Функция getPdo() не найдена в pg_master.php');
        }

        $pdo = getPdo();

        if (!$pdo instanceof PDO) {
            throw new RuntimeException('getPdo() должен вернуть объект PDO');
        }

        /**
         * Настраиваем PDO, чтобы ошибки были нормальными исключениями.
         */
        $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);

        /**
         * По умолчанию получаем строки как ассоциативные массивы.
         *
         * То есть:
         * $row['name']
         *
         * а не:
         * $row[0]
         */
        $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);

        self::$pdo = $pdo;

        return self::$pdo;
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
     *
     * Например:
     * SELECT COUNT(*) FROM table
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
     * Выполнить INSERT / UPDATE / DELETE.
     *
     * Возвращает количество затронутых строк.
     */
    public static function execute(string $sql, array $params = []): int
    {
        return self::query($sql, $params)->rowCount();
    }
}


---

Шаг 2. Создаём /local/mvc/Core/Model.php

Теперь сделаем базовую модель.

Создай файл:

/local/mvc/Core/Model.php

Код:

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * Model
 *
 * Это базовый класс для моделей.
 *
 * Потом от него будут наследоваться:
 *
 * Site
 * Page
 * UserAccess
 * Application
 */
abstract class Model
{
    /**
     * Имя таблицы.
     *
     * Например:
     * protected static string $table = 'sitebuilder.sites';
     */
    protected static string $table = '';

    /**
     * Главный ключ таблицы.
     *
     * Обычно id.
     */
    protected static string $primaryKey = 'id';

    /**
     * Получить имя таблицы.
     */
    protected static function table(): string
    {
        if (static::$table === '') {
            throw new RuntimeException('У модели не указана таблица: ' . static::class);
        }

        return static::$table;
    }

    /**
     * Получить все записи.
     */
    public static function all(int $limit = 100): array
    {
        $limit = max(1, min($limit, 500));

        $sql = 'SELECT * FROM ' . static::table()
            . ' ORDER BY ' . static::$primaryKey . ' DESC'
            . ' LIMIT ' . $limit;

        return Db::fetchAll($sql);
    }

    /**
     * Найти одну запись по ID.
     */
    public static function find(int|string $id): ?array
    {
        $sql = 'SELECT * FROM ' . static::table()
            . ' WHERE ' . static::$primaryKey . ' = :id'
            . ' LIMIT 1';

        return Db::fetchOne($sql, [
            'id' => $id,
        ]);
    }

    /**
     * Посчитать количество записей.
     */
    public static function count(): int
    {
        $sql = 'SELECT COUNT(*) FROM ' . static::table();

        return (int)Db::value($sql);
    }

    /**
     * Удалить запись по ID.
     */
    public static function deleteById(int|string $id): int
    {
        $sql = 'DELETE FROM ' . static::table()
            . ' WHERE ' . static::$primaryKey . ' = :id';

        return Db::execute($sql, [
            'id' => $id,
        ]);
    }
}

Что такое Model простыми словами

Model — это родитель для таблиц.

Например, потом сделаем:

class Site extends Model
{
    protected static string $table = 'sitebuilder.sites';
}

И сможем писать:

$sites = Site::all();
$count = Site::count();
$site = Site::find(5);

То есть модель — это удобная обёртка над таблицей.


---

Шаг 3. Создаём тестовый контроллер базы

Создай файл:

/local/mvc/Controllers/DbController.php

Код:

<?php

namespace Local\Mvc\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Db;
use Local\Mvc\Core\Response;
use Throwable;

/**
 * DbController
 *
 * Контроллер для проверки подключения к PostgreSQL.
 */
class DbController extends Controller
{
    /**
     * Проверка базы.
     */
    public function test(): Response
    {
        try {
            $row = Db::fetchOne("
                SELECT
                    current_database() AS database_name,
                    current_schema() AS schema_name,
                    now() AS server_time
            ");

            return $this->render('db/test', [
                'title' => 'Проверка базы данных',
                'row' => $row,
                'error' => null,
            ]);
        } catch (Throwable $e) {
            return $this->render('db/test', [
                'title' => 'Ошибка подключения к базе',
                'row' => null,
                'error' => $e->getMessage(),
            ]);
        }
    }

    /**
     * JSON-проверка базы.
     */
    public function ping(): Response
    {
        try {
            $row = Db::fetchOne("
                SELECT
                    current_database() AS database_name,
                    current_schema() AS schema_name,
                    now() AS server_time
            ");

            return $this->success([
                'db' => $row,
            ]);
        } catch (Throwable $e) {
            return $this->error('DB_ERROR', [
                'message' => $e->getMessage(),
            ], 500);
        }
    }
}


---

Шаг 4. Создаём View для проверки базы

Создай папку:

/local/mvc/Views/db/

Создай файл:

/local/mvc/Views/db/test.php

Код:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Проверка базы данных') ?>
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
            Подключение к PostgreSQL работает.
        </p>

        <div class="mvc-info">
            <b>Информация от базы:</b>

            <ol>
                <li>
                    База:
                    <span class="mvc-code">
                        <?= htmlspecialcharsbx($row['database_name'] ?? '') ?>
                    </span>
                </li>

                <li>
                    Схема:
                    <span class="mvc-code">
                        <?= htmlspecialcharsbx($row['schema_name'] ?? '') ?>
                    </span>
                </li>

                <li>
                    Время сервера:
                    <span class="mvc-code">
                        <?= htmlspecialcharsbx($row['server_time'] ?? '') ?>
                    </span>
                </li>
            </ol>
        </div>
    <?php endif; ?>
</div>


---

Шаг 5. Обновляем routes.php

Замени файл:

/local/mvc/routes.php

на такой:

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


---

Шаг 6. Проверяем

Открой HTML-страницу проверки:

https://bitrix24-stage.gaz.ru/local/mvc/db/test

Если всё хорошо, увидишь:

Подключение к PostgreSQL работает.

Потом проверь JSON:

https://bitrix24-stage.gaz.ru/local/mvc/db/ping

Должен быть примерно такой ответ:

{
    "ok": true,
    "data": {
        "db": {
            "database_name": "...",
            "schema_name": "public",
            "server_time": "..."
        }
    }
}


---

Что мы сейчас построили

Теперь путь такой:

Браузер
  ↓
/local/mvc/db/test
  ↓
urlrewrite.php
  ↓
index.php
  ↓
Request
  ↓
Router
  ↓
DbController
  ↓
Db
  ↓
PostgreSQL
  ↓
View
  ↓
Layout
  ↓
Response
  ↓
Браузер

Главное простыми словами

Db — это дверь в базу.

Db::fetchAll(...)
Db::fetchOne(...)
Db::value(...)
Db::execute(...)

Model — это будущая удобная работа с конкретными таблицами.

Например:

Site::all()
Site::find(5)
Site::count()

Но пока мы только подготовили основу.


---

Если будет ошибка

Если /local/mvc/db/test покажет ошибку:

Файл подключения к PostgreSQL не найден

значит путь к pg_master.php другой.

Если покажет:

Функция getPdo() не найдена

значит в твоём файле функция называется иначе.

Если покажет:

SQLSTATE...

значит файл подключился, но сама база вернула ошибку.

Следующим шагом сделаем первую настоящую модель, например DemoItem, и научимся выводить список записей из таблицы через MVC.