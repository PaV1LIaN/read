Да, это нормально. Сейчас сайт пишет сюда:

192.168.7.110 → база bx → схема mvc → таблица mvc_demo_notes

А ты смотришь:

ProjectsDB → схема mvc → mvc_demo_notes

Это две разные базы.

Сейчас наш Core\Db работает через подключение Битрикса:

Bitrix\Main\Application::getConnection()

Поэтому он пишет в базу bx.

Если ты хочешь, чтобы Note писал именно в:

ProjectsDB.mvc.mvc_demo_notes

тогда надо сделать как в Laravel: у моделей может быть разное подключение.

Например:

User → bitrix
Note → projects


---

Что делаем

Сделаем так:

protected static string $connection = 'bitrix';

А у Note:

protected static string $connection = 'projects';

Тогда:

User будет читать b_user из базы Битрикса bx
Note будет писать в ProjectsDB


---

1. Замени /local/mvc/Core/Db.php

<?php

namespace Local\Mvc\Core;

use Bitrix\Main\Application;
use PDO;
use RuntimeException;

class Db
{
    private static array $pdoConnections = [];

    /**
     * SELECT много строк.
     */
    public static function fetchAll(string $sql, array $params = [], ?string $connection = null): array
    {
        $connection = $connection ?: self::defaultConnection();

        if ($connection === 'bitrix') {
            $result = Application::getConnection()->query(self::replaceNamedParamsForBitrix($sql, $params));
            return $result->fetchAll();
        }

        $stmt = self::pdo($connection)->prepare($sql);
        self::bindPdoParams($stmt, $params);
        $stmt->execute();

        return $stmt->fetchAll(PDO::FETCH_ASSOC);
    }

    /**
     * SELECT одна строка.
     */
    public static function fetchOne(string $sql, array $params = [], ?string $connection = null): ?array
    {
        $connection = $connection ?: self::defaultConnection();

        if ($connection === 'bitrix') {
            $result = Application::getConnection()->query(self::replaceNamedParamsForBitrix($sql, $params));
            $row = $result->fetch();

            return $row ?: null;
        }

        $stmt = self::pdo($connection)->prepare($sql);
        self::bindPdoParams($stmt, $params);
        $stmt->execute();

        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        return $row ?: null;
    }

    /**
     * SELECT одно значение.
     */
    public static function value(string $sql, array $params = [], ?string $connection = null): mixed
    {
        $connection = $connection ?: self::defaultConnection();

        if ($connection === 'bitrix') {
            $result = Application::getConnection()->query(self::replaceNamedParamsForBitrix($sql, $params));
            $row = $result->fetch();

            if (!$row) {
                return null;
            }

            return reset($row);
        }

        $stmt = self::pdo($connection)->prepare($sql);
        self::bindPdoParams($stmt, $params);
        $stmt->execute();

        return $stmt->fetchColumn();
    }

    /**
     * INSERT / UPDATE / DELETE.
     */
    public static function execute(string $sql, array $params = [], ?string $connection = null): bool
    {
        $connection = $connection ?: self::defaultConnection();

        if ($connection === 'bitrix') {
            Application::getConnection()->queryExecute(self::replaceNamedParamsForBitrix($sql, $params));
            return true;
        }

        $stmt = self::pdo($connection)->prepare($sql);
        self::bindPdoParams($stmt, $params);

        return $stmt->execute();
    }

    private static function defaultConnection(): string
    {
        return (string)Config::get('database.default', 'bitrix');
    }

    private static function pdo(string $connection): PDO
    {
        if (isset(self::$pdoConnections[$connection])) {
            return self::$pdoConnections[$connection];
        }

        $connections = Config::get('database.connections', []);

        if (!is_array($connections) || empty($connections[$connection])) {
            throw new RuntimeException('DB_CONNECTION_NOT_CONFIGURED: ' . $connection);
        }

        $config = $connections[$connection];
        $driver = (string)($config['driver'] ?? '');

        if ($driver === 'pg_master') {
            $file = $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

            if (!is_file($file)) {
                throw new RuntimeException('PG_MASTER_FILE_NOT_FOUND: ' . $file);
            }

            require_once $file;

            if (function_exists('getPdo')) {
                $pdo = getPdo();
            } elseif (function_exists('getPDO')) {
                $pdo = getPDO();
            } else {
                throw new RuntimeException('PG_MASTER_GETPDO_NOT_FOUND');
            }

            if (!$pdo instanceof PDO) {
                throw new RuntimeException('PG_MASTER_DID_NOT_RETURN_PDO');
            }

            $pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
            $pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);

            $schema = trim((string)($config['schema'] ?? ''));

            if ($schema !== '') {
                $pdo->exec('SET search_path TO ' . self::safeSchema($schema) . ', public');
            }

            self::$pdoConnections[$connection] = $pdo;

            return $pdo;
        }

        throw new RuntimeException('DB_DRIVER_NOT_SUPPORTED: ' . $driver);
    }

    private static function bindPdoParams(\PDOStatement $stmt, array $params): void
    {
        foreach ($params as $key => $value) {
            $name = ':' . ltrim((string)$key, ':');
            $stmt->bindValue($name, $value);
        }
    }

    /**
     * Для Bitrix connection пока оставляем простую замену параметров.
     * В наших Bitrix-запросах используются безопасные значения через QueryBuilder.
     */
    private static function replaceNamedParamsForBitrix(string $sql, array $params): string
    {
        if (empty($params)) {
            return $sql;
        }

        $connection = Application::getConnection();

        foreach ($params as $key => $value) {
            $placeholder = ':' . ltrim((string)$key, ':');

            if ($value === null) {
                $escaped = 'NULL';
            } elseif (is_int($value) || is_float($value)) {
                $escaped = (string)$value;
            } else {
                $escaped = "'" . $connection->getSqlHelper()->forSql((string)$value) . "'";
            }

            $sql = str_replace($placeholder, $escaped, $sql);
        }

        return $sql;
    }

    private static function safeSchema(string $schema): string
    {
        if (!preg_match('/^[a-zA-Z_][a-zA-Z0-9_]*$/', $schema)) {
            throw new RuntimeException('BAD_SCHEMA_NAME: ' . $schema);
        }

        return $schema;
    }
}


---

2. Обнови /local/mvc/Core/Model.php

Добавь свойство подключения:

protected static string $connection = 'bitrix';

И метод query() должен передавать подключение.

Полный файл:

<?php

namespace Local\Mvc\Core;

use RuntimeException;

abstract class Model
{
    protected static string $connection = 'bitrix';

    protected static string $table = '';

    protected static string $primaryKey = 'ID';

    protected static array $fillable = [];

    protected static function table(): string
    {
        if (static::$table === '') {
            throw new RuntimeException('У модели не указана таблица: ' . static::class);
        }

        return static::$table;
    }

    public static function query(): QueryBuilder
    {
        return QueryBuilder::table(static::table(), static::$connection);
    }

    public static function all(int $limit = 100): array
    {
        return static::query()
            ->orderBy(static::$primaryKey, 'desc')
            ->limit($limit)
            ->get();
    }

    public static function find(int|string $id): ?array
    {
        return static::query()
            ->where(static::$primaryKey, $id)
            ->first();
    }

    public static function count(): int
    {
        return static::query()->count();
    }

    public static function create(array $data): bool
    {
        return static::query()->insert(static::onlyFillable($data));
    }

    public static function updateById(int|string $id, array $data): bool
    {
        return static::query()
            ->where(static::$primaryKey, $id)
            ->update(static::onlyFillable($data));
    }

    public static function deleteById(int|string $id): bool
    {
        return static::query()
            ->where(static::$primaryKey, $id)
            ->delete();
    }

    protected static function onlyFillable(array $data): array
    {
        if (empty(static::$fillable)) {
            return $data;
        }

        $result = [];

        foreach (static::$fillable as $field) {
            if (array_key_exists($field, $data)) {
                $result[$field] = $data[$field];
            }
        }

        return $result;
    }
}


---

3. В /local/mvc/Core/QueryBuilder.php добавь connection

В начале класса добавь свойство:

private string $connection;

Конструктор замени на:

public function __construct(string $table, string $connection = 'bitrix')
{
    $this->table = $this->safeTable($table);
    $this->connection = $connection;
}

Метод table() замени на:

public static function table(string $table, string $connection = 'bitrix'): self
{
    return new self($table, $connection);
}

Теперь найди в QueryBuilder.php вызовы Db и добавь туда $this->connection.

Было:

return Db::fetchAll($this->toSql(), $this->bindings);

Стало:

return Db::fetchAll($this->toSql(), $this->bindings, $this->connection);

Было:

return Db::fetchOne($clone->toSql(), $clone->bindings);

Стало:

return Db::fetchOne($clone->toSql(), $clone->bindings, $this->connection);

Было:

return (int)Db::value($sql, $this->bindings);

Стало:

return (int)Db::value($sql, $this->bindings, $this->connection);

Было:

return Db::execute($sql, $bindings);

Стало:

return Db::execute($sql, $bindings, $this->connection);

Было:

return Db::execute($sql, $this->bindings);

Стало:

return Db::execute($sql, $this->bindings, $this->connection);


---

4. Обнови /local/mvc_demo/config.php

Добавь блок database:

'database' => [
    'default' => 'bitrix',

    'connections' => [
        'bitrix' => [
            'driver' => 'bitrix',
        ],

        'projects' => [
            'driver' => 'pg_master',
            'schema' => 'mvc',
        ],
    ],
],

Итог примерно такой:

<?php

return [
    'app' => [
        'name' => 'MVC Demo',
        'description' => 'Тестовый проект на общем MVC-фреймворке',
    ],

    'debug' => true,

    'log' => [
        'file' => __DIR__ . '/logs/app.log',
    ],

    'database' => [
        'default' => 'bitrix',

        'connections' => [
            'bitrix' => [
                'driver' => 'bitrix',
            ],

            'projects' => [
                'driver' => 'pg_master',
                'schema' => 'mvc',
            ],
        ],
    ],

    'providers' => [
        \Local\MvcDemo\Providers\AppServiceProvider::class,
    ],

    'roles' => [
        'resolver' => \Local\MvcDemo\Services\DemoRoleResolver::class,

        'hierarchy' => [
            'viewer' => 10,
            'editor' => 20,
            'admin' => 30,
            'owner' => 40,
        ],
    ],
];


---

5. Обнови /local/mvc_demo/Models/Note.php

Главное — добавь:

protected static string $connection = 'projects';

Полный файл:

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Model;

class Note extends Model
{
    protected static string $connection = 'projects';

    protected static string $table = 'mvc.mvc_demo_notes';

    protected static string $primaryKey = 'id';

    protected static array $fillable = [
        'title',
        'body',
        'created_at',
        'updated_at',
    ];

    public static function latest(int $limit = 20): array
    {
        $rows = self::query()
            ->orderBy('id', 'desc')
            ->limit($limit)
            ->get();

        return array_map([self::class, 'normalize'], $rows);
    }

    public static function normalize(array $row): array
    {
        return [
            'id' => $row['id'] ?? $row['ID'] ?? null,
            'title' => $row['title'] ?? $row['TITLE'] ?? '',
            'body' => $row['body'] ?? $row['BODY'] ?? '',
            'created_at' => $row['created_at'] ?? $row['CREATED_AT'] ?? '',
            'updated_at' => $row['updated_at'] ?? $row['UPDATED_AT'] ?? '',
        ];
    }
}


---

6. Проверь таблицу в ProjectsDB

В pgAdmin именно в:

ProjectsDB → Schemas → mvc

выполни:

CREATE TABLE IF NOT EXISTS mvc.mvc_demo_notes (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Создай новую заметку.

Теперь она должна появиться именно здесь:

ProjectsDB → mvc → mvc_demo_notes


---

Важно: старые заметки останутся в базе bx.mvc.mvc_demo_notes. Новые после этих правок должны пойти в ProjectsDB.mvc.mvc_demo_notes.