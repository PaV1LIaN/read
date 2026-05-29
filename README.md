Идём дальше. Сейчас сделаем очень Laravel-like вещь — Route Model Binding.

В Laravel можно писать так:

public function edit(Note $note)
{
    //
}

А фреймворк сам понимает:

В URL пришёл id заметки.
Надо найти Note по этому id.
И передать готовую модель в контроллер.

Сейчас у нас так:

public function edit(string $id): Response
{
    $note = Note::findNormalized((int)$id);
}

А хотим так:

public function edit(Note $note): Response
{
    //
}


---

1. Создай /local/mvc/Core/ModelNotFoundException.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

class ModelNotFoundException extends RuntimeException
{
    public function __construct(
        private string $model,
        private int|string $id
    ) {
        parent::__construct('Модель не найдена: ' . $model . ' ID=' . $id);
    }

    public function model(): string
    {
        return $this->model;
    }

    public function id(): int|string
    {
        return $this->id;
    }
}


---

2. Замени /local/mvc/Core/Model.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

abstract class Model
{
    protected static string $connection = 'bitrix';

    protected static string $table = '';

    protected static string $primaryKey = 'ID';

    protected static array $fillable = [];

    protected static string $factory = '';

    protected static bool $timestamps = false;

    protected static string $createdAtColumn = 'created_at';

    protected static string $updatedAtColumn = 'updated_at';

    protected array $attributes = [];

    public function __construct(array $attributes = [])
    {
        $this->attributes = $attributes;
    }

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

    public static function factory(): Factory
    {
        if (static::$factory === '') {
            throw new RuntimeException('У модели не указана factory: ' . static::class);
        }

        if (!class_exists(static::$factory)) {
            throw new RuntimeException('Factory не найдена: ' . static::$factory);
        }

        $factory = new static::$factory();

        if (!$factory instanceof Factory) {
            throw new RuntimeException('Factory должна наследоваться от Factory: ' . static::$factory);
        }

        return $factory;
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

    /**
     * Найти запись и вернуть объект модели.
     *
     * Это нужно для Route Model Binding:
     * public function edit(Note $note)
     */
    public static function findModel(int|string $id): ?static
    {
        $row = static::find($id);

        if (!$row) {
            return null;
        }

        return new static($row);
    }

    public static function count(): int
    {
        return static::query()->count();
    }

    public static function create(array $data): bool
    {
        $data = static::applyCreateTimestamps($data);

        return static::query()->insert(static::onlyFillable($data));
    }

    public static function updateById(int|string $id, array $data): bool
    {
        $data = static::applyUpdateTimestamps($data);

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

    /**
     * Обновить текущую модель.
     */
    public function update(array $data): bool
    {
        return static::updateById($this->getKey(), $data);
    }

    /**
     * Удалить текущую модель.
     */
    public function delete(): bool
    {
        return static::deleteById($this->getKey());
    }

    public function getKey(): int|string
    {
        return $this->getAttribute(static::$primaryKey)
            ?? $this->getAttribute(strtolower(static::$primaryKey))
            ?? $this->getAttribute(strtoupper(static::$primaryKey))
            ?? 0;
    }

    public function getAttribute(string $key, mixed $default = null): mixed
    {
        if (array_key_exists($key, $this->attributes)) {
            return $this->attributes[$key];
        }

        $upper = strtoupper($key);

        if (array_key_exists($upper, $this->attributes)) {
            return $this->attributes[$upper];
        }

        $lower = strtolower($key);

        if (array_key_exists($lower, $this->attributes)) {
            return $this->attributes[$lower];
        }

        return $default;
    }

    public function __get(string $key): mixed
    {
        return $this->getAttribute($key);
    }

    public function toArray(): array
    {
        return $this->attributes;
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

    protected static function applyCreateTimestamps(array $data): array
    {
        if (!static::$timestamps) {
            return $data;
        }

        $now = static::freshTimestamp();

        if (!array_key_exists(static::$createdAtColumn, $data)) {
            $data[static::$createdAtColumn] = $now;
        }

        if (!array_key_exists(static::$updatedAtColumn, $data)) {
            $data[static::$updatedAtColumn] = $now;
        }

        return $data;
    }

    protected static function applyUpdateTimestamps(array $data): array
    {
        if (!static::$timestamps) {
            return $data;
        }

        $data[static::$updatedAtColumn] = static::freshTimestamp();

        return $data;
    }

    protected static function freshTimestamp(): string
    {
        return date('Y-m-d H:i:s');
    }
}


---

3. Обнови /local/mvc/Core/Container.php

Найди в методе resolveParameters() кусок:

if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
    $object = $this->make($type->getName());

    if ($object instanceof FormRequest) {
        $object->validateResolved();
    }

    $dependencies[] = $object;
    continue;
}

Замени его на:

if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
    $className = $type->getName();

    /**
     * Laravel-like Route Model Binding.
     *
     * Если метод контроллера просит модель:
     * edit(Note $note)
     *
     * А в маршруте есть {id},
     * то контейнер сам делает Note::findModel($id).
     */
    if (is_subclass_of($className, Model::class) && array_key_exists('id', $parameters)) {
        $model = $className::findModel($parameters['id']);

        if (!$model instanceof Model) {
            throw new ModelNotFoundException($className, $parameters['id']);
        }

        $dependencies[] = $model;
        continue;
    }

    $object = $this->make($className);

    if ($object instanceof FormRequest) {
        $object->validateResolved();
    }

    $dependencies[] = $object;
    continue;
}


---

4. Обнови /local/mvc/Core/ErrorHandler.php

В методе:

public static function renderThrowable(Throwable $e): void

в самое начало добавь:

if ($e instanceof ModelNotFoundException) {
    self::renderModelNotFoundException($e);
    return;
}

Должно быть примерно так:

public static function renderThrowable(Throwable $e): void
{
    if ($e instanceof ModelNotFoundException) {
        self::renderModelNotFoundException($e);
        return;
    }

    if ($e instanceof ValidationException) {
        self::renderValidationException($e);
        return;
    }

    // дальше старый код
}

Теперь в этот же класс добавь метод:

private static function renderModelNotFoundException(ModelNotFoundException $e): void
{
    self::log($e->getMessage(), $e->getFile(), $e->getLine());

    if (self::wantsJson()) {
        Response::json([
            'ok' => false,
            'error' => 'MODEL_NOT_FOUND',
            'details' => [
                'message' => 'Запись не найдена.',
                'model' => $e->model(),
                'id' => $e->id(),
            ],
        ], 404)->send();

        return;
    }

    Response::html(
        '<h1>404</h1>'
        . '<p>Запись не найдена.</p>'
        . '<pre>'
        . htmlspecialchars($e->model() . ' ID=' . $e->id())
        . '</pre>',
        404
    )->send();
}


---

5. Обнови /local/mvc_demo/Models/Note.php

Добавим метод normalized() для объекта модели.

Полный файл:

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Model;
use Local\MvcDemo\Database\Factories\NoteFactory;

class Note extends Model
{
    protected static string $connection = 'projects';

    protected static string $table = 'mvc.mvc_demo_notes';

    protected static string $primaryKey = 'id';

    protected static string $factory = NoteFactory::class;

    protected static bool $timestamps = true;

    protected static string $createdAtColumn = 'created_at';

    protected static string $updatedAtColumn = 'updated_at';

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

    public static function findNormalized(int $id): ?array
    {
        $row = self::find($id);

        if (!$row) {
            return null;
        }

        return self::normalize($row);
    }

    public function normalized(): array
    {
        return self::normalize($this->toArray());
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

6. Обнови /local/mvc_demo/Controllers/NoteController.php

Теперь методы edit, update, destroy будут получать Note $note, а не string $id.

Полный файл:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Models\Note;
use Local\MvcDemo\Requests\StoreNoteRequest;
use Local\MvcDemo\Requests\UpdateNoteRequest;

class NoteController extends Controller
{
    public function index(): Response
    {
        return $this->render('notes/index', [
            'title' => 'Заметки',
            'notes' => Note::latest(20),
        ]);
    }

    public function store(StoreNoteRequest $request): Response
    {
        $data = $request->validated();

        Note::create([
            'title' => $data['title'],
            'body' => $data['body'],
        ]);

        Flash::success('Заметка создана.');

        return redirect()->route('notes.index');
    }

    public function edit(Note $note): Response
    {
        return $this->render('notes/edit', [
            'title' => 'Редактирование заметки',
            'note' => $note->normalized(),
        ]);
    }

    public function update(Note $note, UpdateNoteRequest $request): Response
    {
        $data = $request->validated();

        $note->update([
            'title' => $data['title'],
            'body' => $data['body'],
        ]);

        Flash::success('Заметка обновлена.');

        return redirect()->route('notes.index');
    }

    public function destroy(Note $note): Response
    {
        $note->delete();

        Flash::success('Заметка удалена.');

        return redirect()->route('notes.index');
    }

    public function factory(): Response
    {
        Note::factory()
            ->count(5)
            ->create();

        Flash::success('Factory создала 5 тестовых заметок.');

        return redirect()->route('notes.index');
    }
}


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь:

/local/mvc_demo/notes

Сценарии:

1. Нажми "Изменить".
2. Измени заметку.
3. Удали заметку.
4. Открой несуществующую:
   /local/mvc_demo/notes/999999/edit

Для несуществующей заметки должен быть нормальный 404, а не 500.


---

Что мы сделали

Раньше контроллер сам искал модель:

public function edit(string $id): Response
{
    $note = Note::findNormalized((int)$id);
}

Теперь Laravel-like:

public function edit(Note $note): Response
{
    //
}

Простыми словами:

Router нашёл id из URL.
Container увидел, что контроллеру нужен Note.
Container сделал Note::findModel(id).
Если запись найдена — передал её в контроллер.
Если не найдена — отдал 404.

Это очень важная часть Laravel — Route Model Binding.