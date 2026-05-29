Идём дальше. Добавим в наш Laravel-like QueryBuilder:

orWhere()
whereLike()
orWhereLike()
whereRaw()
orWhereRaw()

Чтобы поиск можно было писать ближе к Laravel:

User::query()
    ->whereLike('LOGIN', $search)
    ->orWhereLike('EMAIL', $search)
    ->orWhereLike('NAME', $search)
    ->get();


---

1. Замени /local/mvc/Core/QueryBuilder.php

<?php

namespace Local\Mvc\Core;

/**
 * QueryBuilder
 *
 * Laravel-like построитель SELECT-запросов.
 *
 * Поддерживает:
 * - select
 * - where
 * - orWhere
 * - whereLike
 * - orWhereLike
 * - whereRaw
 * - orWhereRaw
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
        return $this->addBasicWhere('AND', $column, $operator, $value, func_num_args());
    }

    /**
     * orWhere('LOGIN', 'admin')
     * orWhere('ID', '>', 10)
     */
    public function orWhere(string $column, mixed $operator = null, mixed $value = null): self
    {
        return $this->addBasicWhere('OR', $column, $operator, $value, func_num_args());
    }

    /**
     * whereLike('LOGIN', 'admin')
     *
     * Сам добавит проценты:
     * LOGIN LIKE %admin%
     */
    public function whereLike(string $column, string $value): self
    {
        return $this->where($column, 'LIKE', '%' . $value . '%');
    }

    /**
     * orWhereLike('EMAIL', 'admin')
     */
    public function orWhereLike(string $column, string $value): self
    {
        return $this->orWhere($column, 'LIKE', '%' . $value . '%');
    }

    /**
     * whereRaw('CAST(ID AS CHAR) LIKE :q', ['q' => '%10%'])
     *
     * Использовать аккуратно.
     * Это нужно для сложных условий, которые QueryBuilder пока не умеет собрать сам.
     */
    public function whereRaw(string $sql, array $bindings = []): self
    {
        return $this->addRawWhere('AND', $sql, $bindings);
    }

    public function orWhereRaw(string $sql, array $bindings = []): self
    {
        return $this->addRawWhere('OR', $sql, $bindings);
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

        $clone = clone $this;

        $items = $clone
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

    private function addBasicWhere(
        string $boolean,
        string $column,
        mixed $operator,
        mixed $value,
        int $argumentCount
    ): self {
        if ($argumentCount === 2) {
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
            'type' => 'basic',
            'boolean' => $boolean,
            'column' => $this->safeColumn($column),
            'operator' => $operator,
            'binding' => $binding,
        ];

        $this->bindings[$binding] = $value;

        return $this;
    }

    private function addRawWhere(string $boolean, string $sql, array $bindings = []): self
    {
        $sql = trim($sql);

        if ($sql === '') {
            return $this;
        }

        $this->wheres[] = [
            'type' => 'raw',
            'boolean' => $boolean,
            'sql' => $sql,
        ];

        foreach ($bindings as $key => $value) {
            $key = ltrim((string)$key, ':');

            $this->bindings[$key] = $value;
        }

        return $this;
    }

    private function compileWheres(): string
    {
        $parts = [];

        foreach ($this->wheres as $index => $where) {
            $boolean = strtoupper((string)($where['boolean'] ?? 'AND'));

            if ($index === 0) {
                $boolean = '';
            }

            if (($where['type'] ?? '') === 'raw') {
                $piece = '(' . $where['sql'] . ')';
            } else {
                $piece = $where['column'] . ' ' . $where['operator'] . ' :' . $where['binding'];
            }

            $parts[] = trim($boolean . ' ' . $piece);
        }

        return implode(' ', $parts);
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

2. Обнови /local/mvc_demo/Models/User.php

Теперь перепишем поиск через QueryBuilder.

Полностью замени файл:

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Model;
use Local\Mvc\Core\QueryBuilder;

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
        return self::baseQuery()
            ->orderBy('ID', 'desc')
            ->limit($limit)
            ->get();
    }

    public static function latestPage(int $limit = 10, int $offset = 0): array
    {
        return self::baseQuery()
            ->orderBy('ID', 'desc')
            ->limit($limit)
            ->offset($offset)
            ->get();
    }

    public static function findForAdmin(int $id): ?array
    {
        return self::baseQuery()
            ->where('ID', $id)
            ->first();
    }

    public static function countSearch(string $search = ''): int
    {
        $search = trim($search);

        if ($search === '') {
            return self::count();
        }

        return self::searchQuery($search)->count();
    }

    public static function searchPage(string $search = '', int $limit = 10, int $offset = 0): array
    {
        $search = trim($search);

        if ($search === '') {
            return self::latestPage($limit, $offset);
        }

        return self::searchQuery($search)
            ->orderBy('ID', 'desc')
            ->limit($limit)
            ->offset($offset)
            ->get();
    }

    /**
     * Базовый набор колонок.
     */
    private static function baseQuery(): QueryBuilder
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
            ]);
    }

    /**
     * Поисковый запрос.
     *
     * Здесь специально используем whereRaw(),
     * потому что условие с OR удобнее собрать одним блоком.
     */
    private static function searchQuery(string $search): QueryBuilder
    {
        return self::baseQuery()
            ->whereRaw("
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

3. Обнови тест queryBuilderTest в HomeController

Найди метод:

public function queryBuilderTest(): Response

Замени на:

public function queryBuilderTest(): Response
{
    $search = trim((string)request('q', ''));

    $query = User::query()
        ->select(['ID', 'LOGIN', 'EMAIL', 'ACTIVE'])
        ->where('ACTIVE', 'Y')
        ->orderBy('ID', 'desc')
        ->limit(5);

    if ($search !== '') {
        $query
            ->whereLike('LOGIN', $search)
            ->orWhereLike('EMAIL', $search);
    }

    $users = $query->get();

    return response()->json([
        'ok' => true,
        'data' => [
            'message' => 'QueryBuilder работает',
            'search' => $search,
            'sql' => $query->toSql(),
            'bindings' => $query->bindings(),
            'users' => $users,
        ],
    ]);
}


---

4. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь обычный запрос:

/local/mvc_demo/query-builder-test

Потом с поиском:

/local/mvc_demo/query-builder-test?q=admin

Также проверь старые страницы:

/local/mvc_demo/admin/users
/local/mvc_demo/admin/users?q=admin
/local/mvc_demo/api/users?q=admin


---

Важный момент

В тестовом методе:

->where('ACTIVE', 'Y')
->whereLike('LOGIN', $search)
->orWhereLike('EMAIL', $search)

SQL получится примерно такой:

WHERE ACTIVE = :p1 AND LOGIN LIKE :p2 OR EMAIL LIKE :p3

Это пока простая версия без группировки условий.

То есть логически это читается так:

ACTIVE = Y AND LOGIN LIKE q
OR EMAIL LIKE q

Позже мы добавим группировки:

->where(function ($query) {
    $query->whereLike('LOGIN', 'admin')
          ->orWhereLike('EMAIL', 'admin');
})

И тогда будет как в Laravel:

WHERE ACTIVE = Y AND (LOGIN LIKE q OR EMAIL LIKE q)


---

Что мы сделали

Теперь наш QueryBuilder стал ближе к Laravel:

User::query()
    ->where('ACTIVE', 'Y')
    ->whereLike('LOGIN', 'admin')
    ->orWhereLike('EMAIL', 'admin')
    ->orderBy('ID', 'desc')
    ->limit(10)
    ->get();

И модель User стала чище:

return self::searchQuery($search)
    ->orderBy('ID', 'desc')
    ->limit($limit)
    ->offset($offset)
    ->get();

Главная мысль:

QueryBuilder — это не сама база.
Это помощник, который собирает SQL.
Db потом выполняет этот SQL.

Следующий шаг — добавить группировку where, чтобы писать условия почти как в Laravel.