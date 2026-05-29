Отлично. Дальше делаем Laravel-like Query Builder.

Цель — чтобы можно было писать похоже на Laravel:

User::query()
    ->select(['ID', 'LOGIN', 'EMAIL'])
    ->where('ACTIVE', 'Y')
    ->orderBy('ID', 'desc')
    ->limit(10)
    ->get();

Сейчас у нас в модели SQL пишется руками. А Query Builder — это промежуточный слой, который помогает собирать SQL аккуратнее.


---

1. Создай /local/mvc/Core/QueryBuilder.php

<?php

namespace Local\Mvc\Core;

/**
 * QueryBuilder
 *
 * Laravel-like построитель SELECT-запросов.
 *
 * Пока простой:
 * - select
 * - where
 * - orderBy
 * - limit
 * - offset
 * - get
 * - first
 * - count
 * - paginate
 */
class QueryBuilder
{
    private string $table;

    private array $select = ['*'];

    private array $wheres = [];

    private array $orders = [];

    private ?int $limit = null;

    private ?int $offset = null;

    private array $bindings = [];

    private int $bindingIndex = 0;

    public function __construct(string $table)
    {
        $this->table = $this->safeTable($table);
    }

    public static function table(string $table): self
    {
        return new self($table);
    }

    public function select(array|string $columns): self
    {
        if (is_string($columns)) {
            $columns = [$columns];
        }

        $this->select = array_map(
            fn ($column) => $this->safeColumn((string)$column),
            $columns
        );

        return $this;
    }

    /**
     * where('ACTIVE', 'Y')
     * where('ID', '>', 10)
     */
    public function where(string $column, mixed $operator = null, mixed $value = null): self
    {
        if (func_num_args() === 2) {
            $value = $operator;
            $operator = '=';
        }

        $operator = strtoupper(trim((string)$operator));

        $allowed = ['=', '!=', '<>', '>', '<', '>=', '<=', 'LIKE'];

        if (!in_array($operator, $allowed, true)) {
            throw new \InvalidArgumentException('QUERY_BUILDER_BAD_OPERATOR: ' . $operator);
        }

        $binding = $this->nextBindingName();

        $this->wheres[] = [
            'column' => $this->safeColumn($column),
            'operator' => $operator,
            'binding' => $binding,
        ];

        $this->bindings[$binding] = $value;

        return $this;
    }

    public function orderBy(string $column, string $direction = 'asc'): self
    {
        $direction = strtolower(trim($direction));

        if (!in_array($direction, ['asc', 'desc'], true)) {
            $direction = 'asc';
        }

        $this->orders[] = [
            'column' => $this->safeColumn($column),
            'direction' => strtoupper($direction),
        ];

        return $this;
    }

    public function limit(int $limit): self
    {
        $this->limit = max(1, min($limit, 500));

        return $this;
    }

    public function offset(int $offset): self
    {
        $this->offset = max(0, $offset);

        return $this;
    }

    public function get(): array
    {
        return Db::fetchAll($this->toSql(), $this->bindings);
    }

    public function first(): ?array
    {
        $clone = clone $this;
        $clone->limit(1);

        return Db::fetchOne($clone->toSql(), $clone->bindings);
    }

    public function count(): int
    {
        $sql = 'SELECT COUNT(*) FROM ' . $this->table;

        if (!empty($this->wheres)) {
            $sql .= ' WHERE ' . $this->compileWheres();
        }

        return (int)Db::value($sql, $this->bindings);
    }

    public function paginate(int $page = 1, int $perPage = 10): array
    {
        $total = $this->count();

        $paginator = new Paginator($total, $page, $perPage);

        $items = $this
            ->limit($paginator->perPage())
            ->offset($paginator->offset())
            ->get();

        return [
            'items' => $items,
            'pagination' => $paginator->toArray(),
        ];
    }

    public function toSql(): string
    {
        $sql = 'SELECT ' . implode(', ', $this->select);
        $sql .= ' FROM ' . $this->table;

        if (!empty($this->wheres)) {
            $sql .= ' WHERE ' . $this->compileWheres();
        }

        if (!empty($this->orders)) {
            $parts = [];

            foreach ($this->orders as $order) {
                $parts[] = $order['column'] . ' ' . $order['direction'];
            }

            $sql .= ' ORDER BY ' . implode(', ', $parts);
        }

        if ($this->limit !== null) {
            $sql .= ' LIMIT ' . $this->limit;
        }

        if ($this->offset !== null) {
            $sql .= ' OFFSET ' . $this->offset;
        }

        return $sql;
    }

    public function bindings(): array
    {
        return $this->bindings;
    }

    private function compileWheres(): string
    {
        $parts = [];

        foreach ($this->wheres as $where) {
            $parts[] = $where['column'] . ' ' . $where['operator'] . ' :' . $where['binding'];
        }

        return implode(' AND ', $parts);
    }

    private function nextBindingName(): string
    {
        $this->bindingIndex++;

        return 'p' . $this->bindingIndex;
    }

    private function safeTable(string $table): string
    {
        $table = trim($table);

        if (!preg_match('/^[a-zA-Z_][a-zA-Z0-9_]*(\.[a-zA-Z_][a-zA-Z0-9_]*)?$/', $table)) {
            throw new \InvalidArgumentException('QUERY_BUILDER_BAD_TABLE: ' . $table);
        }

        return $table;
    }

    private function safeColumn(string $column): string
    {
        $column = trim($column);

        if ($column === '*') {
            return '*';
        }

        if (!preg_match('/^[a-zA-Z_][a-zA-Z0-9_]*(\.[a-zA-Z_][a-zA-Z0-9_]*)?$/', $column)) {
            throw new \InvalidArgumentException('QUERY_BUILDER_BAD_COLUMN: ' . $column);
        }

        return $column;
    }
}


---

2. Обнови /local/mvc/Core/Model.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * Model
 *
 * Базовая модель.
 *
 * Похожа на простую Laravel Model:
 *
 * User::query()
 * User::find(1)
 * User::count()
 */
abstract class Model
{
    protected static string $table = '';

    protected static string $primaryKey = 'ID';

    protected static function table(): string
    {
        if (static::$table === '') {
            throw new RuntimeException('У модели не указана таблица: ' . static::class);
        }

        return static::$table;
    }

    public static function query(): QueryBuilder
    {
        return QueryBuilder::table(static::table());
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

    public static function deleteById(int|string $id): bool
    {
        return Db::execute(
            'DELETE FROM ' . static::table() . '
             WHERE ' . static::$primaryKey . ' = :id',
            [
                'id' => $id,
            ]
        );
    }
}


---

3. Обнови /local/mvc_demo/Models/User.php

Можно оставить старые методы, но перепишем часть через query().

Полностью замени файл:

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Db;
use Local\Mvc\Core\Model;

/**
 * User
 *
 * Модель пользователя Битрикса.
 */
class User extends Model
{
    protected static string $table = 'b_user';

    protected static string $primaryKey = 'ID';

    public static function latest(int $limit = 10): array
    {
        return self::query()
            ->select([
                'ID',
                'LOGIN',
                'NAME',
                'LAST_NAME',
                'SECOND_NAME',
                'EMAIL',
                'ACTIVE',
                'DATE_REGISTER',
                'LAST_LOGIN',
            ])
            ->orderBy('ID', 'desc')
            ->limit($limit)
            ->get();
    }

    public static function latestPage(int $limit = 10, int $offset = 0): array
    {
        return self::query()
            ->select([
                'ID',
                'LOGIN',
                'NAME',
                'LAST_NAME',
                'SECOND_NAME',
                'EMAIL',
                'ACTIVE',
                'DATE_REGISTER',
                'LAST_LOGIN',
            ])
            ->orderBy('ID', 'desc')
            ->limit($limit)
            ->offset($offset)
            ->get();
    }

    public static function findForAdmin(int $id): ?array
    {
        return self::query()
            ->select([
                'ID',
                'LOGIN',
                'NAME',
                'LAST_NAME',
                'SECOND_NAME',
                'EMAIL',
                'ACTIVE',
                'DATE_REGISTER',
                'LAST_LOGIN',
            ])
            ->where('ID', $id)
            ->first();
    }

    /**
     * Здесь пока оставим ручной SQL,
     * потому что поиск OR сложнее.
     *
     * OR-where добавим следующим шагом.
     */
    public static function countSearch(string $search = ''): int
    {
        $search = trim($search);

        if ($search === '') {
            return self::count();
        }

        return (int)Db::value("
            SELECT COUNT(*)
            FROM b_user
            WHERE
                CAST(ID AS CHAR) LIKE :q
                OR LOGIN LIKE :q
                OR NAME LIKE :q
                OR LAST_NAME LIKE :q
                OR SECOND_NAME LIKE :q
                OR EMAIL LIKE :q
        ", [
            'q' => '%' . $search . '%',
        ]);
    }

    public static function searchPage(string $search = '', int $limit = 10, int $offset = 0): array
    {
        $search = trim($search);
        $limit = max(1, min($limit, 100));
        $offset = max(0, $offset);

        if ($search === '') {
            return self::latestPage($limit, $offset);
        }

        return Db::fetchAll("
            SELECT
                ID,
                LOGIN,
                NAME,
                LAST_NAME,
                SECOND_NAME,
                EMAIL,
                ACTIVE,
                DATE_REGISTER,
                LAST_LOGIN
            FROM b_user
            WHERE
                CAST(ID AS CHAR) LIKE :q
                OR LOGIN LIKE :q
                OR NAME LIKE :q
                OR LAST_NAME LIKE :q
                OR SECOND_NAME LIKE :q
                OR EMAIL LIKE :q
            ORDER BY ID DESC
            LIMIT {$limit} OFFSET {$offset}
        ", [
            'q' => '%' . $search . '%',
        ]);
    }

    public static function fullName(array $user): string
    {
        $lastName = trim((string)($user['LAST_NAME'] ?? ''));
        $name = trim((string)($user['NAME'] ?? ''));
        $secondName = trim((string)($user['SECOND_NAME'] ?? ''));

        $fullName = trim($lastName . ' ' . $name . ' ' . $secondName);

        if ($fullName !== '') {
            return $fullName;
        }

        return (string)($user['LOGIN'] ?? '');
    }
}


---

4. Добавь тест в HomeController

Открой:

/local/mvc_demo/Controllers/HomeController.php

Сверху добавь:

use Local\MvcDemo\Models\User;

Внутрь класса добавь метод:

public function queryBuilderTest(): Response
{
    $users = User::query()
        ->select(['ID', 'LOGIN', 'EMAIL', 'ACTIVE'])
        ->where('ACTIVE', 'Y')
        ->orderBy('ID', 'desc')
        ->limit(5)
        ->get();

    return response()->json([
        'ok' => true,
        'data' => [
            'message' => 'QueryBuilder работает',
            'sql_example' => User::query()
                ->select(['ID', 'LOGIN'])
                ->where('ACTIVE', 'Y')
                ->orderBy('ID', 'desc')
                ->limit(5)
                ->toSql(),
            'users' => $users,
        ],
    ]);
}


---

5. Добавь маршрут в /local/mvc_demo/routes.php

Route::get('/query-builder-test', [HomeController::class, 'queryBuilderTest'])
    ->name('query.builder.test');


---

6. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/query-builder-test

Должен быть JSON:

{
  "ok": true,
  "data": {
    "message": "QueryBuilder работает",
    "sql_example": "SELECT ID, LOGIN FROM b_user WHERE ACTIVE = :p1 ORDER BY ID DESC LIMIT 5",
    "users": [...]
  }
}

Потом обязательно проверь старые страницы:

/local/mvc_demo/admin/users
/local/mvc_demo/admin/dashboard
/local/mvc_demo/api/users


---

Что мы сделали

Раньше модель писала SQL руками:

Db::fetchAll("
    SELECT ID, LOGIN
    FROM b_user
    ORDER BY ID DESC
    LIMIT 10
");

Теперь можно писать ближе к Laravel:

User::query()
    ->select(['ID', 'LOGIN'])
    ->where('ACTIVE', 'Y')
    ->orderBy('ID', 'desc')
    ->limit(10)
    ->get();

Главная мысль:

Model знает таблицу.
QueryBuilder помогает собирать запрос.
Db выполняет запрос.

Это ещё не Eloquent, но уже очень хороший мостик к Laravel.

Следующий шаг — добавить в QueryBuilder orWhere() и whereLike(), чтобы переписать поиск пользователей без ручного SQL.