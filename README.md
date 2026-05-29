Идём дальше. Сейчас сделаем Laravel-like автоматические timestamps.

В Laravel модель сама умеет заполнять поля:

created_at
updated_at

То есть в контроллере не надо каждый раз писать:

'created_at' => date('Y-m-d H:i:s'),
'updated_at' => date('Y-m-d H:i:s'),

Хотим так:

Note::create([
    'title' => $data['title'],
    'body' => $data['body'],
]);

А модель сама добавит даты.


---

1. Замени /local/mvc/Core/Model.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

abstract class Model
{
    protected static string $connection = 'bitrix';

    protected static string $table = '';

    protected static string $primaryKey = 'ID';

    protected static array $fillable = [];

    /**
     * Laravel-like timestamps.
     *
     * Если true, модель сама заполняет:
     * created_at
     * updated_at
     */
    protected static bool $timestamps = false;

    protected static string $createdAtColumn = 'created_at';

    protected static string $updatedAtColumn = 'updated_at';

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

2. Обнови /local/mvc_demo/Models/Note.php

Тут главное добавить:

protected static bool $timestamps = true;

Полный файл:

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Model;

class Note extends Model
{
    protected static string $connection = 'projects';

    protected static string $table = 'mvc.mvc_demo_notes';

    protected static string $primaryKey = 'id';

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

3. Обнови /local/mvc_demo/Controllers/NoteController.php

Теперь убираем ручное заполнение дат.

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

    public function edit(string $id): Response
    {
        $note = Note::findNormalized((int)$id);

        if (!$note) {
            return Response::html(
                '<h1>404</h1><p>Заметка не найдена.</p>',
                404
            );
        }

        return $this->render('notes/edit', [
            'title' => 'Редактирование заметки',
            'note' => $note,
        ]);
    }

    public function update(string $id, UpdateNoteRequest $request): Response
    {
        $note = Note::findNormalized((int)$id);

        if (!$note) {
            return Response::html(
                '<h1>404</h1><p>Заметка не найдена.</p>',
                404
            );
        }

        $data = $request->validated();

        Note::updateById((int)$id, [
            'title' => $data['title'],
            'body' => $data['body'],
        ]);

        Flash::success('Заметка обновлена.');

        return redirect()->route('notes.index');
    }

    public function destroy(string $id): Response
    {
        Note::deleteById((int)$id);

        Flash::success('Заметка удалена.');

        return redirect()->route('notes.index');
    }
}


---

4. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Проверь:

1. Создай новую заметку.
2. В pgAdmin в ProjectsDB → mvc → mvc_demo_notes должны заполниться created_at и updated_at.
3. Измени заметку.
4. updated_at должен обновиться.


---

Что мы сделали

Раньше контроллер сам ставил даты:

'created_at' => date('Y-m-d H:i:s'),
'updated_at' => date('Y-m-d H:i:s'),

Теперь это делает модель:

protected static bool $timestamps = true;

Простыми словами:

Controller говорит: "создай заметку".
Model сама добавляет created_at и updated_at.

Это ещё один шаг к Laravel, потому что в Laravel модель тоже сама управляет created_at и updated_at.