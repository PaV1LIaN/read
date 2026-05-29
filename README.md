Отлично, идём дальше. Сейчас добавим группировку where, как в Laravel.

Сейчас у нас может получиться не совсем правильная логика:

User::query()
    ->where('ACTIVE', 'Y')
    ->whereLike('LOGIN', $search)
    ->orWhereLike('EMAIL', $search);

SQL будет примерно такой:

WHERE ACTIVE = 'Y' AND LOGIN LIKE '%admin%' OR EMAIL LIKE '%admin%'

Проблема: OR EMAIL может сработать отдельно от ACTIVE.

А правильно так:

WHERE ACTIVE = 'Y' AND (
    LOGIN LIKE '%admin%'
    OR EMAIL LIKE '%admin%'
)

В Laravel это пишется так:

User::query()
    ->where('ACTIVE', 'Y')
    ->where(function ($query) use ($search) {
        $query->whereLike('LOGIN', $search)
              ->orWhereLike('EMAIL', $search);
    });

Сейчас сделаем так же.


---

1. Замени /local/mvc/Core/QueryBuilder.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

use Closure;
use InvalidArgumentException;

/**
 * QueryBuilder
 *
 * Laravel-like построитель SELECT-запросов.
 *
 * Теперь умеет группировку:
 *
 * ->where(function ($query) {
 *     $query->where('A', 1)
 *           ->orWhere('B', 2);
 * })
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
     * Варианты:
     *
     * where('ACTIVE', 'Y')
     * where('ID', '>', 10)
     * where(function ($query) { ... })
     */
    public function where(string|callable $column, mixed $operator = null, mixed $value = null): self
    {
        if (is_callable($column)) {
            return $this->addNestedWhere('AND', $column);
        }

        return $this->addBasicWhere('AND', $column, $operator, $value, func_num_args());
    }

    /**
     * orWhere('LOGIN', 'admin')
     * orWhere(function ($query) { ... })
     */
    public function orWhere(string|callable $column, mixed $operator = null, mixed $value = null): self
    {
        if (is_callable($column)) {
            return $this->addNestedWhere('OR', $column);
        }

        return $this->addBasicWhere('OR', $column, $operator, $value, func_num_args());
    }

    public function whereLike(string $column, string $value): self
    {
        return $this->where($column, 'LIKE', '%' . $value . '%');
    }

    public function orWhereLike(string $column, string $value): self
    {
        return $this->orWhere($column, 'LIKE', '%' . $value . '%');
    }

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
            throw new InvalidArgumentException('QUERY_BUILDER_BAD_OPERATOR: ' . $operator);
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

    /**
     * Группировка условий.
     *
     * Пример:
     * ->where(function ($query) {
     *     $query->whereLike('LOGIN', 'admin')
     *           ->orWhereLike('EMAIL', 'admin');
     * })
     *
     * SQL:
     * AND (LOGIN LIKE :p2 OR EMAIL LIKE :p3)
     */
    private function addNestedWhere(string $boolean, callable $callback): self
    {
        $nested = new self($this->table);

        $callback($nested);

        if (empty($nested->wheres)) {
            return $this;
        }

        $nestedSql = $nested->compileWheres();

        /**
         * У nested builder свои параметры :p1, :p2.
         * Чтобы они не конфликтовали с родительскими,
         * переименуем их в новые параметры родителя.
         */
        foreach ($nested->bindings as $oldKey => $value) {
            $newKey = $this->nextBindingName();

            $nestedSql = preg_replace(
                '/:' . preg_quote((string)$oldKey, '/') . '\b/',
                ':' . $newKey,
                $nestedSql
            );

            $this->bindings[$newKey] = $value;
        }

        $this->wheres[] = [
            'type' => 'nested',
            'boolean' => $boolean,
            'sql' => $nestedSql,
        ];

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
            } elseif (($where['type'] ?? '') === 'nested') {
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
            throw new InvalidArgumentException('QUERY_BUILDER_BAD_TABLE: ' . $table);
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
            throw new InvalidArgumentException('QUERY_BUILDER_BAD_COLUMN: ' . $column);
        }

        return $column;
    }
}


---

2. Обнови /local/mvc_demo/Models/User.php

Теперь можно переписать поиск без сырого SQL.

Найди метод:

private static function searchQuery(string $search): QueryBuilder

Замени его на:

private static function searchQuery(string $search): QueryBuilder
{
    return self::baseQuery()
        ->where(function (QueryBuilder $query) use ($search) {
            $query
                ->whereRaw('CAST(ID AS CHAR) LIKE :q', [
                    'q' => '%' . $search . '%',
                ])
                ->orWhereLike('LOGIN', $search)
                ->orWhereLike('NAME', $search)
                ->orWhereLike('LAST_NAME', $search)
                ->orWhereLike('SECOND_NAME', $search)
                ->orWhereLike('EMAIL', $search);
        });
}

Мы всё ещё используем whereRaw() только для CAST(ID AS CHAR), потому что это специфичная SQL-штука. Остальное уже через builder.


---

3. Обнови queryBuilderTest в HomeController

Замени метод:

public function queryBuilderTest(): Response

на:

public function queryBuilderTest(): Response
{
    $search = trim((string)request('q', ''));

    $query = User::query()
        ->select(['ID', 'LOGIN', 'EMAIL', 'ACTIVE'])
        ->where('ACTIVE', 'Y');

    if ($search !== '') {
        $query->where(function (QueryBuilder $query) use ($search) {
            $query
                ->whereLike('LOGIN', $search)
                ->orWhereLike('EMAIL', $search);
        });
    }

    $query
        ->orderBy('ID', 'desc')
        ->limit(5);

    $users = $query->get();

    return response()->json([
        'ok' => true,
        'data' => [
            'message' => 'QueryBuilder с группировкой работает',
            'search' => $search,
            'sql' => $query->toSql(),
            'bindings' => $query->bindings(),
            'users' => $users,
        ],
    ]);
}

И проверь, что сверху в HomeController.php есть:

use Local\Mvc\Core\QueryBuilder;


---

4. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/query-builder-test?q=admin

В ответе SQL должен быть примерно такой:

SELECT ID, LOGIN, EMAIL, ACTIVE
FROM b_user
WHERE ACTIVE = :p1 AND (LOGIN LIKE :p2 OR EMAIL LIKE :p3)
ORDER BY ID DESC
LIMIT 5

Это главное.

Потом проверь:

/local/mvc_demo/admin/users?q=admin
/local/mvc_demo/api/users?q=admin


---

Что мы сделали

Теперь можно писать почти как в Laravel:

User::query()
    ->where('ACTIVE', 'Y')
    ->where(function (QueryBuilder $query) use ($search) {
        $query->whereLike('LOGIN', $search)
              ->orWhereLike('EMAIL', $search);
    })
    ->orderBy('ID', 'desc')
    ->get();

Простыми словами:

where(...)        — обычное условие
orWhere(...)      — условие через OR
where(function)   — скобки вокруг группы условий

Это уже намного ближе к Laravel Query Builder.