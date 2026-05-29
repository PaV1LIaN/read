Идём дальше. После Migrations и Seeders в Laravel обычно идут Factories.

Простыми словами:

Migration — создаёт таблицу.
Seeder — запускает заполнение.
Factory — умеет создавать тестовые данные.

В Laravel это выглядит примерно так:

Note::factory()->count(3)->create();

Сделаем похожее.


---

1. Создай /local/mvc/Core/Factory.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * Factory
 *
 * Laravel-like фабрика тестовых данных.
 *
 * Пример:
 * Note::factory()->count(3)->create();
 */
abstract class Factory
{
    /**
     * Класс модели, для которой работает factory.
     *
     * Например:
     * protected string $model = Note::class;
     */
    protected string $model = '';

    protected int $count = 1;

    protected array $state = [];

    public static function new(): static
    {
        return new static();
    }

    /**
     * Описание одной записи.
     */
    abstract public function definition(): array;

    /**
     * Сколько записей создать.
     */
    public function count(int $count): static
    {
        $new = clone $this;
        $new->count = max(1, min($count, 1000));

        return $new;
    }

    /**
     * Перезаписать часть данных.
     *
     * Пример:
     * Note::factory()->state([
     *     'title' => 'Моё название'
     * ])->create();
     */
    public function state(array $state): static
    {
        $new = clone $this;
        $new->state = array_merge($new->state, $state);

        return $new;
    }

    /**
     * Просто подготовить данные, но не сохранять в БД.
     */
    public function make(array $attributes = []): array
    {
        $items = [];

        for ($i = 0; $i < $this->count; $i++) {
            $items[] = $this->raw($attributes);
        }

        return $this->count === 1 ? $items[0] : $items;
    }

    /**
     * Создать записи в БД.
     */
    public function create(array $attributes = []): array
    {
        $model = $this->modelClass();

        $created = [];

        for ($i = 0; $i < $this->count; $i++) {
            $data = $this->raw($attributes);

            $model::create($data);

            $created[] = $data;
        }

        return $created;
    }

    /**
     * Данные одной записи.
     */
    protected function raw(array $attributes = []): array
    {
        return array_merge(
            $this->definition(),
            $this->state,
            $attributes
        );
    }

    protected function modelClass(): string
    {
        if ($this->model === '') {
            throw new RuntimeException('FACTORY_MODEL_NOT_SET: ' . static::class);
        }

        if (!class_exists($this->model)) {
            throw new RuntimeException('FACTORY_MODEL_CLASS_NOT_FOUND: ' . $this->model);
        }

        if (!is_subclass_of($this->model, Model::class)) {
            throw new RuntimeException('FACTORY_MODEL_MUST_EXTEND_MODEL: ' . $this->model);
        }

        return $this->model;
    }
}


---

2. Обнови /local/mvc/Core/Model.php

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

    /**
     * Factory-класс модели.
     *
     * Например:
     * protected static string $factory = NoteFactory::class;
     */
    protected static string $factory = '';

    /**
     * Laravel-like timestamps.
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

    /**
     * Laravel-like factory.
     *
     * Пример:
     * Note::factory()->count(3)->create();
     */
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

3. Создай папку factories

/local/mvc_demo/Database/Factories/


---

4. Создай /local/mvc_demo/Database/Factories/NoteFactory.php

<?php

namespace Local\MvcDemo\Database\Factories;

use Local\Mvc\Core\Factory;
use Local\MvcDemo\Models\Note;

class NoteFactory extends Factory
{
    protected string $model = Note::class;

    public function definition(): array
    {
        $titles = [
            'Тестовая заметка',
            'Заметка из factory',
            'Laravel-like MVC',
            'Проверка CRUD',
            'Работа с PostgreSQL',
        ];

        $bodies = [
            'Эта запись создана через NoteFactory.',
            'Factory нужна для генерации тестовых данных.',
            'Такой подход похож на Laravel factories.',
            'Seeder запускает factory, а factory создаёт данные.',
            'Это удобно для проверки таблиц и страниц.',
        ];

        return [
            'title' => $titles[array_rand($titles)] . ' #' . random_int(1000, 9999),
            'body' => $bodies[array_rand($bodies)],
        ];
    }
}


---

5. Обнови /local/mvc_demo/Models/Note.php

Полностью замени файл:

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

6. Обнови /local/mvc_demo/Database/Seeders/DemoNotesSeeder.php

Полностью замени файл:

<?php

namespace Local\MvcDemo\Database\Seeders;

use Local\Mvc\Core\Seeder;
use Local\MvcDemo\Models\Note;

class DemoNotesSeeder extends Seeder
{
    public function run(): void
    {
        /**
         * Чтобы не плодить одинаковые записи каждый раз,
         * добавляем тестовые заметки только если таблица пустая.
         */
        if (Note::count() > 0) {
            return;
        }

        Note::factory()
            ->count(3)
            ->create();
    }
}

Вот это уже очень похоже на Laravel:

Note::factory()->count(3)->create();


---

7. Добавим страницу теста factory

Открой:

/local/mvc_demo/Controllers/NoteController.php

Добавь внутрь класса метод:

public function factory(): Response
{
    Note::factory()
        ->count(5)
        ->create();

    Flash::success('Factory создала 5 тестовых заметок.');

    return redirect()->route('notes.index');
}


---

8. Добавь маршрут в /local/mvc_demo/routes.php

Добавь рядом с notes:

Route::post('/notes/factory', [NoteController::class, 'factory'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.factory');

Важно: поставь этот маршрут выше Route::resource('/notes', ...), чтобы /notes/factory не конфликтовал с /notes/{id}.

Должно быть примерно так:

Route::post('/notes/factory', [NoteController::class, 'factory'])
    ->middleware(['auth', 'admin', 'csrf'])
    ->name('notes.factory');

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

9. Добавь кнопку в /local/mvc_demo/Views/notes/index.php

После формы создания заметки добавь ещё один блок:

<div class="mvc-info">
    <form method="post" action="<?= e(route('notes.factory')) ?>">
        <?= csrf_field() ?>

        <button
            type="submit"
            style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#7c3aed;color:#fff;font-weight:600;cursor:pointer;"
        >
            Создать 5 заметок через Factory
        </button>
    </form>
</div>


---

10. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Нажми:

Создать 5 заметок через Factory

Должны появиться 5 случайных заметок.


---

Что мы сделали

Теперь у нас есть Laravel-like factory:

Note::factory()
    ->count(5)
    ->create();

Простыми словами:

Factory — это шаблон тестовой записи.
Seeder — решает, когда и сколько таких записей создать.
Model — сохраняет запись в таблицу.

В Laravel ты потом увидишь почти то же самое:

User::factory()->count(10)->create();

Теперь эта идея тебе уже будет знакома.