Идём дальше. Раз у нас появились миграции, следующий Laravel-like шаг — сделать Schema Builder.

В Laravel миграция обычно выглядит не так:

$this->statement("CREATE TABLE ...");

А красиво:

Schema::create('notes', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('body')->nullable();
    $table->timestamps();
});

Сейчас сделаем похожее.


---

1. Создай /local/mvc/Core/ColumnDefinition.php

<?php

namespace Local\Mvc\Core;

class ColumnDefinition
{
    private bool $nullable = false;

    public function __construct(
        private string $name,
        private string $type,
        private bool $primary = false
    ) {}

    public function nullable(): self
    {
        $this->nullable = true;

        return $this;
    }

    public function toSql(): string
    {
        $sql = $this->safeColumn($this->name) . ' ' . $this->type;

        if ($this->primary) {
            return $sql . ' PRIMARY KEY';
        }

        $sql .= $this->nullable ? ' NULL' : ' NOT NULL';

        return $sql;
    }

    private function safeColumn(string $column): string
    {
        $column = trim($column);

        if (!preg_match('/^[a-zA-Z_][a-zA-Z0-9_]*$/', $column)) {
            throw new \InvalidArgumentException('BAD_COLUMN_NAME: ' . $column);
        }

        return $column;
    }
}


---

2. Создай /local/mvc/Core/Blueprint.php

<?php

namespace Local\Mvc\Core;

/**
 * Blueprint
 *
 * Laravel-like описание таблицы.
 *
 * Пример:
 * $table->id();
 * $table->string('title');
 * $table->text('body')->nullable();
 * $table->timestamps();
 */
class Blueprint
{
    private array $columns = [];

    public function id(string $name = 'id'): ColumnDefinition
    {
        return $this->addColumn(new ColumnDefinition($name, 'BIGSERIAL', true));
    }

    public function string(string $name, int $length = 255): ColumnDefinition
    {
        $length = max(1, min($length, 1000));

        return $this->addColumn(new ColumnDefinition($name, 'VARCHAR(' . $length . ')'));
    }

    public function text(string $name): ColumnDefinition
    {
        return $this->addColumn(new ColumnDefinition($name, 'TEXT'));
    }

    public function integer(string $name): ColumnDefinition
    {
        return $this->addColumn(new ColumnDefinition($name, 'INTEGER'));
    }

    public function bigInteger(string $name): ColumnDefinition
    {
        return $this->addColumn(new ColumnDefinition($name, 'BIGINT'));
    }

    public function timestamp(string $name): ColumnDefinition
    {
        return $this->addColumn(new ColumnDefinition($name, 'TIMESTAMP'));
    }

    public function timestamps(): void
    {
        $this->timestamp('created_at')->nullable();
        $this->timestamp('updated_at')->nullable();
    }

    public function toSqlColumns(): array
    {
        return array_map(
            static fn (ColumnDefinition $column) => $column->toSql(),
            $this->columns
        );
    }

    private function addColumn(ColumnDefinition $column): ColumnDefinition
    {
        $this->columns[] = $column;

        return $column;
    }
}


---

3. Создай /local/mvc/Core/SchemaBuilder.php

<?php

namespace Local\Mvc\Core;

use Closure;
use RuntimeException;

/**
 * SchemaBuilder
 *
 * Laravel-like создание/удаление таблиц.
 */
class SchemaBuilder
{
    public function __construct(
        private ?string $connection = null
    ) {}

    public function connection(string $connection): self
    {
        return new self($connection);
    }

    public function create(string $table, callable $callback): void
    {
        $connection = $this->connectionName();

        $this->ensureSchemaExists($connection);

        $blueprint = new Blueprint();

        $callback($blueprint);

        $columns = $blueprint->toSqlColumns();

        if (empty($columns)) {
            throw new RuntimeException('SCHEMA_CREATE_NO_COLUMNS: ' . $table);
        }

        $sql = 'CREATE TABLE IF NOT EXISTS '
            . $this->qualifiedTable($table, $connection)
            . " (\n    "
            . implode(",\n    ", $columns)
            . "\n)";

        Db::execute($sql, [], $connection);
    }

    public function dropIfExists(string $table): void
    {
        $connection = $this->connectionName();

        Db::execute(
            'DROP TABLE IF EXISTS ' . $this->qualifiedTable($table, $connection),
            [],
            $connection
        );
    }

    private function connectionName(): string
    {
        return $this->connection ?: (string)Config::get('database.default', 'bitrix');
    }

    private function ensureSchemaExists(string $connection): void
    {
        $schema = $this->schemaForConnection($connection);

        if ($schema === '') {
            return;
        }

        Db::execute(
            'CREATE SCHEMA IF NOT EXISTS ' . $this->safeIdentifier($schema),
            [],
            $connection
        );
    }

    private function qualifiedTable(string $table, string $connection): string
    {
        $table = trim($table);

        /**
         * Если уже передали schema.table — оставляем как есть после проверки.
         */
        if (str_contains($table, '.')) {
            return $this->safeQualifiedTable($table);
        }

        $schema = $this->schemaForConnection($connection);

        if ($schema !== '') {
            return $this->safeIdentifier($schema) . '.' . $this->safeIdentifier($table);
        }

        return $this->safeIdentifier($table);
    }

    private function schemaForConnection(string $connection): string
    {
        return trim((string)Config::get('database.connections.' . $connection . '.schema', ''));
    }

    private function safeQualifiedTable(string $table): string
    {
        $parts = explode('.', $table, 2);

        if (count($parts) !== 2) {
            throw new RuntimeException('BAD_TABLE_NAME: ' . $table);
        }

        return $this->safeIdentifier($parts[0]) . '.' . $this->safeIdentifier($parts[1]);
    }

    private function safeIdentifier(string $value): string
    {
        $value = trim($value);

        if (!preg_match('/^[a-zA-Z_][a-zA-Z0-9_]*$/', $value)) {
            throw new RuntimeException('BAD_IDENTIFIER: ' . $value);
        }

        return $value;
    }
}


---

4. Создай facade /local/mvc/Support/Facades/Schema.php

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\SchemaBuilder;

/**
 * Schema
 *
 * Laravel-like facade для миграций.
 *
 * Пример:
 * Schema::connection('projects')->create(...)
 */
class Schema extends Facade
{
    protected static function accessor(): string
    {
        return SchemaBuilder::class;
    }
}


---

5. Зарегистрируй SchemaBuilder в /local/mvc/Core/App.php

Открой:

/local/mvc/Core/App.php

Найди блок, где регистрируются сервисы:

$container->singleton(\Local\Mvc\Core\LogManager::class, \Local\Mvc\Core\LogManager::class);
$container->singleton(\Local\Mvc\Core\ConfigManager::class, \Local\Mvc\Core\ConfigManager::class);
$container->singleton(\Local\Mvc\Core\ResponseFactory::class, \Local\Mvc\Core\ResponseFactory::class);
$container->singleton(\Local\Mvc\Core\Redirector::class, \Local\Mvc\Core\Redirector::class);

Добавь ниже:

$container->singleton(\Local\Mvc\Core\SchemaBuilder::class, \Local\Mvc\Core\SchemaBuilder::class);

Итог:

$container->singleton(\Local\Mvc\Core\LogManager::class, \Local\Mvc\Core\LogManager::class);
$container->singleton(\Local\Mvc\Core\ConfigManager::class, \Local\Mvc\Core\ConfigManager::class);
$container->singleton(\Local\Mvc\Core\ResponseFactory::class, \Local\Mvc\Core\ResponseFactory::class);
$container->singleton(\Local\Mvc\Core\Redirector::class, \Local\Mvc\Core\Redirector::class);
$container->singleton(\Local\Mvc\Core\SchemaBuilder::class, \Local\Mvc\Core\SchemaBuilder::class);


---

6. Обнови миграцию заметок

Файл:

/local/mvc_demo/Database/Migrations/2026_05_29_000001_create_mvc_demo_notes_table.php

Полностью замени:

<?php

use Local\Mvc\Core\Blueprint;
use Local\Mvc\Core\Migration;
use Local\Mvc\Support\Facades\Schema;

return new class extends Migration {
    protected string $connection = 'projects';

    public function up(): void
    {
        Schema::connection('projects')->create('mvc_demo_notes', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('body')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::connection('projects')->dropIfExists('mvc_demo_notes');
    }
};

Обрати внимание:

Schema::connection('projects')->create('mvc_demo_notes', ...)

Мы не пишем mvc.mvc_demo_notes, потому что схема mvc уже указана в config:

'projects' => [
    'driver' => 'pg_master',
    'schema' => 'mvc',
],


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/migrations

Если миграция уже применена, она останется в статусе Применена.

Чтобы проверить новый Schema Builder, можно сделать так:

1. Нажми "Откатить последнюю пачку".
2. Таблица mvc_demo_notes удалится.
3. Нажми "Запустить миграции".
4. Таблица создастся уже через Schema Builder.

Важно: rollback удалит заметки, потому что down() делает DROP TABLE.


---

Что мы сделали

Раньше миграция была с сырым SQL:

$this->statement("
    CREATE TABLE IF NOT EXISTS mvc.mvc_demo_notes (...)
");

Теперь Laravel-like:

Schema::connection('projects')->create('mvc_demo_notes', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('body')->nullable();
    $table->timestamps();
});

Главная мысль:

Migration — говорит, что сделать.
Schema — создаёт или удаляет таблицу.
Blueprint — описывает колонки таблицы.

Это уже очень похоже на Laravel migration system.