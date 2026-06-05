Идём дальше. Сделаем Laravel-like Soft Deletes.

В Laravel запись можно не удалять физически из таблицы, а “мягко удалить”:

deleted_at = дата удаления

То есть запись остаётся в базе, но обычные запросы её больше не показывают.


---

Что хотим получить

Сейчас кнопка Удалить делает настоящий DELETE FROM.

После этого шага будет так:

$note->delete();

но в базе произойдёт:

UPDATE mvc.mvc_demo_notes
SET deleted_at = NOW()
WHERE id = ...

А обычный список:

Note::latest()

будет показывать только записи, где:

deleted_at IS NULL


---

1. Создай новую миграцию

Файл:

/local/mvc_demo/Database/Migrations/2026_05_29_000002_add_deleted_at_to_mvc_demo_notes_table.php

Код:

<?php

use Local\Mvc\Core\Migration;

return new class extends Migration {
    protected string $connection = 'projects';

    public function up(): void
    {
        $this->statement("
            ALTER TABLE mvc.mvc_demo_notes
            ADD COLUMN IF NOT EXISTS deleted_at TIMESTAMP NULL
        ");
    }

    public function down(): void
    {
        $this->statement("
            ALTER TABLE mvc.mvc_demo_notes
            DROP COLUMN IF EXISTS deleted_at
        ");
    }
};


---

2. Обнови /local/mvc/Core/QueryBuilder.php

Добавим whereNull() и whereNotNull().

Внутрь класса добавь методы рядом с whereLike():

public function whereNull(string $column): self
{
    return $this->addNullWhere('AND', $column, true);
}

public function orWhereNull(string $column): self
{
    return $this->addNullWhere('OR', $column, true);
}

public function whereNotNull(string $column): self
{
    return $this->addNullWhere('AND', $column, false);
}

public function orWhereNotNull(string $column): self
{
    return $this->addNullWhere('OR', $column, false);
}

Теперь ниже, рядом с addRawWhere(), добавь private-метод:

private function addNullWhere(string $boolean, string $column, bool $isNull): self
{
    $this->wheres[] = [
        'type' => $isNull ? 'null' : 'not_null',
        'boolean' => $boolean,
        'column' => $this->safeColumn($column),
    ];

    return $this;
}

Теперь в методе compileWheres() найди кусок:

if (($where['type'] ?? '') === 'raw') {
    $piece = '(' . $where['sql'] . ')';
} elseif (($where['type'] ?? '') === 'nested') {
    $piece = '(' . $where['sql'] . ')';
} else {
    $piece = $where['column'] . ' ' . $where['operator'] . ' :' . $where['binding'];
}

Замени на:

if (($where['type'] ?? '') === 'raw') {
    $piece = '(' . $where['sql'] . ')';
} elseif (($where['type'] ?? '') === 'nested') {
    $piece = '(' . $where['sql'] . ')';
} elseif (($where['type'] ?? '') === 'null') {
    $piece = $where['column'] . ' IS NULL';
} elseif (($where['type'] ?? '') === 'not_null') {
    $piece = $where['column'] . ' IS NOT NULL';
} else {
    $piece = $where['column'] . ' ' . $where['operator'] . ' :' . $where['binding'];
}


---

3. Обнови /local/mvc/Core/Model.php

Добавим поддержку soft delete.

Полностью замени файл:

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

    /**
     * Laravel-like soft deletes.
     */
    protected static bool $softDeletes = false;

    protected static string $deletedAtColumn = 'deleted_at';

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
        $query = QueryBuilder::table(static::table(), static::$connection);

        if (static::$softDeletes) {
            $query->whereNull(static::$deletedAtColumn);
        }

        return $query;
    }

    /**
     * Запрос вместе с удалёнными.
     */
    public static function withTrashed(): QueryBuilder
    {
        return QueryBuilder::table(static::table(), static::$connection);
    }

    /**
     * Только мягко удалённые.
     */
    public static function onlyTrashed(): QueryBuilder
    {
        return QueryBuilder::table(static::table(), static::$connection)
            ->whereNotNull(static::$deletedAtColumn);
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
        if (static::$softDeletes) {
            return static::query()
                ->where(static::$primaryKey, $id)
                ->update([
                    static::$deletedAtColumn => static::freshTimestamp(),
                ]);
        }

        return static::query()
            ->where(static::$primaryKey, $id)
            ->delete();
    }

    /**
     * Настоящее физическое удаление.
     */
    public static function forceDeleteById(int|string $id): bool
    {
        return static::withTrashed()
            ->where(static::$primaryKey, $id)
            ->delete();
    }

    /**
     * Восстановить мягко удалённую запись.
     */
    public static function restoreById(int|string $id): bool
    {
        if (!static::$softDeletes) {
            return false;
        }

        return static::withTrashed()
            ->where(static::$primaryKey, $id)
            ->update([
                static::$deletedAtColumn => null,
            ]);
    }

    public function update(array $data): bool
    {
        return static::updateById($this->getKey(), $data);
    }

    public function delete(): bool
    {
        return static::deleteById($this->getKey());
    }

    public function forceDelete(): bool
    {
        return static::forceDeleteById($this->getKey());
    }

    public function restore(): bool
    {
        return static::restoreById($this->getKey());
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

4. Обнови /local/mvc_demo/Models/Note.php

Добавь soft deletes.

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

    protected static bool $softDeletes = true;

    protected static string $createdAtColumn = 'created_at';

    protected static string $updatedAtColumn = 'updated_at';

    protected static string $deletedAtColumn = 'deleted_at';

    protected static array $fillable = [
        'title',
        'body',
        'created_at',
        'updated_at',
        'deleted_at',
    ];

    public static function latest(int $limit = 20): array
    {
        $rows = self::query()
            ->orderBy('id', 'desc')
            ->limit($limit)
            ->get();

        return array_map([self::class, 'normalize'], $rows);
    }

    public static function trashedLatest(int $limit = 20): array
    {
        $rows = self::onlyTrashed()
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
            'deleted_at' => $row['deleted_at'] ?? $row['DELETED_AT'] ?? '',
        ];
    }
}


---

5. Обнови /local/mvc_demo/Controllers/NoteController.php

Добавим страницу корзины и восстановление.

Полностью замени файл:

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
        $this->authorize('viewAny', Note::class);

        return $this->render('notes/index', [
            'title' => 'Заметки',
            'notes' => Note::latest(20),
        ]);
    }

    public function trash(): Response
    {
        $this->authorize('viewAny', Note::class);

        return $this->render('notes/trash', [
            'title' => 'Удалённые заметки',
            'notes' => Note::trashedLatest(20),
        ]);
    }

    public function store(StoreNoteRequest $request): Response
    {
        $this->authorize('create', Note::class);

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
        $this->authorize('update', $note);

        return $this->render('notes/edit', [
            'title' => 'Редактирование заметки',
            'note' => $note->normalized(),
        ]);
    }

    public function update(Note $note, UpdateNoteRequest $request): Response
    {
        $this->authorize('update', $note);

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
        $this->authorize('delete', $note);

        $note->delete();

        Flash::success('Заметка мягко удалена. Её можно восстановить из корзины.');

        return redirect()->route('notes.index');
    }

    public function restore(string $note): Response
    {
        $this->authorize('create', Note::class);

        Note::restoreById((int)$note);

        Flash::success('Заметка восстановлена.');

        return redirect()->route('notes.trash');
    }

    public function forceDelete(string $note): Response
    {
        $this->authorize('delete', Note::class);

        Note::forceDeleteById((int)$note);

        Flash::success('Заметка удалена окончательно.');

        return redirect()->route('notes.trash');
    }

    public function factory(): Response
    {
        $this->authorize('create', Note::class);

        Note::factory()
            ->count(5)
            ->create();

        Flash::success('Factory создала 5 тестовых заметок.');

        return redirect()->route('notes.index');
    }
}


---

6. Создай view /local/mvc_demo/Views/notes/trash.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$canRestoreNote = can('create', \Local\MvcDemo\Models\Note::class);
$canForceDeleteNote = can('delete', \Local\MvcDemo\Models\Note::class);

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= e($title ?? 'Удалённые заметки') ?>
    </h1>

    <p class="mvc-page-text">
        Это корзина. Здесь лежат записи, у которых заполнено
        <span class="mvc-code">deleted_at</span>.
    </p>

    <?php if (!empty($flash)): ?>
        <?php foreach ($flash as $item): ?>
            <div class="mvc-info" style="border-color:#bbf7d0;background:#f0fdf4;color:#166534;">
                <?= e($item['message'] ?? '') ?>
            </div>
        <?php endforeach; ?>
    <?php endif; ?>

    <div class="mvc-info">
        <a href="<?= e(route('notes.index')) ?>">← Назад к заметкам</a>
    </div>

    <div class="mvc-info">
        <?php if (empty($notes)): ?>
            <p style="margin:0;">Корзина пустая.</p>
        <?php else: ?>
            <table style="width:100%;border-collapse:collapse;">
                <thead>
                    <tr>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">ID</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Название</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Удалена</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Действие</th>
                    </tr>
                </thead>

                <tbody>
                    <?php foreach ($notes as $note): ?>
                        <tr>
                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['id'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['title'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['deleted_at'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <div style="display:flex;gap:8px;align-items:center;">
                                    <?php if ($canRestoreNote): ?>
                                        <form method="post" action="<?= e(route('notes.restore', ['note' => (int)($note['id'] ?? 0)])) ?>" style="margin:0;">
                                            <?= csrf_field() ?>

                                            <button
                                                type="submit"
                                                style="padding:6px 10px;border:0;border-radius:8px;background:#16a34a;color:#fff;cursor:pointer;"
                                            >
                                                Восстановить
                                            </button>
                                        </form>
                                    <?php endif; ?>

                                    <?php if ($canForceDeleteNote): ?>
                                        <form method="post" action="<?= e(route('notes.force-delete', ['note' => (int)($note['id'] ?? 0)])) ?>" style="margin:0;">
                                            <?= csrf_field() ?>
                                            <?= method_field('DELETE') ?>

                                            <button
                                                type="submit"
                                                onclick="return confirm('Удалить окончательно?')"
                                                style="padding:6px 10px;border:0;border-radius:8px;background:#dc2626;color:#fff;cursor:pointer;"
                                            >
                                                Удалить навсегда
                                            </button>
                                        </form>
                                    <?php endif; ?>
                                </div>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>
</div>


---

7. Обнови маршруты /local/mvc_demo/routes.php

Важно: эти маршруты должны быть выше Route::resource('/notes', ...).

Добавь рядом с notes.factory:

Route::get('/notes/trash', [NoteController::class, 'trash'])
    ->middleware(['auth'])
    ->name('notes.trash');

Route::post('/notes/{note:\d+}/restore', [NoteController::class, 'restore'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.restore');

Route::delete('/notes/{note:\d+}/force-delete', [NoteController::class, 'forceDelete'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.force-delete');

Итого порядок должен быть такой:

Route::post('/notes/factory', [NoteController::class, 'factory'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.factory');

Route::get('/notes/trash', [NoteController::class, 'trash'])
    ->middleware(['auth'])
    ->name('notes.trash');

Route::post('/notes/{note:\d+}/restore', [NoteController::class, 'restore'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.restore');

Route::delete('/notes/{note:\d+}/force-delete', [NoteController::class, 'forceDelete'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.force-delete');

Route::resource('/notes', NoteController::class, [
    'only' => [
        'index',
        'store',
        'edit',
        'update',
        'destroy',
    ],
    'middleware' => ['csrf'],
]);


---

8. Добавь ссылку на корзину в /local/mvc_demo/Views/notes/index.php

Где-нибудь под заголовком добавь:

<div class="mvc-info">
    <a href="<?= e(route('notes.trash')) ?>">Открыть корзину удалённых заметок</a>
</div>


---

9. Запусти миграцию

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/migrations

Нажми:

Запустить миграции

Должна примениться новая миграция:

2026_05_29_000002_add_deleted_at_to_mvc_demo_notes_table


---

10. Проверка

Открой:

/local/mvc_demo/notes

Проверь:

1. Удали заметку.
2. Она должна исчезнуть из обычного списка.
3. Открой корзину.
4. Там должна быть удалённая заметка.
5. Нажми "Восстановить".
6. Она должна вернуться в обычный список.
7. Снова удали и нажми "Удалить навсегда".
8. Тогда запись физически удалится из БД.


---

Что мы сделали

Теперь модель Note работает почти как Laravel-модель с SoftDeletes:

$note->delete();       // мягкое удаление
$note->restore();      // восстановление
$note->forceDelete();  // удалить навсегда

Note::onlyTrashed();   // только удалённые
Note::withTrashed();   // все, включая удалённые

Главная мысль:

Обычное удаление теперь безопасное.
Запись не исчезает из базы, а попадает в корзину.

Это очень частая практика в Laravel-проектах.