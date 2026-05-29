Идём дальше в Laravel-стиле. Сейчас сделаем запись данных через Model, почти как в Laravel:

Note::create([
    'TITLE' => 'Тест',
    'BODY' => 'Сообщение',
]);

Note::updateById($id, [
    'TITLE' => 'Новое название',
]);

Note::deleteById($id);

Это уже основа для CRUD: создать, показать, изменить, удалить.


---

1. Обнови /local/mvc/Core/QueryBuilder.php

Нам нужно добавить методы:

insert()
update()
delete()

Внутрь класса QueryBuilder, перед методом get(), добавь:

public function insert(array $data): bool
{
    if (empty($data)) {
        return false;
    }

    $columns = [];
    $placeholders = [];
    $bindings = [];

    foreach ($data as $column => $value) {
        $safeColumn = $this->safeColumn((string)$column);
        $binding = $this->nextBindingName();

        $columns[] = $safeColumn;
        $placeholders[] = ':' . $binding;
        $bindings[$binding] = $value;
    }

    $sql = 'INSERT INTO ' . $this->table
        . ' (' . implode(', ', $columns) . ')'
        . ' VALUES (' . implode(', ', $placeholders) . ')';

    return Db::execute($sql, $bindings);
}

public function update(array $data): bool
{
    if (empty($data)) {
        return false;
    }

    $sets = [];
    $bindings = $this->bindings;

    foreach ($data as $column => $value) {
        $safeColumn = $this->safeColumn((string)$column);
        $binding = $this->nextBindingName();

        $sets[] = $safeColumn . ' = :' . $binding;
        $bindings[$binding] = $value;
    }

    $sql = 'UPDATE ' . $this->table
        . ' SET ' . implode(', ', $sets);

    if (!empty($this->wheres)) {
        $sql .= ' WHERE ' . $this->compileWheres();
    }

    return Db::execute($sql, $bindings);
}

public function delete(): bool
{
    $sql = 'DELETE FROM ' . $this->table;

    if (!empty($this->wheres)) {
        $sql .= ' WHERE ' . $this->compileWheres();
    }

    return Db::execute($sql, $this->bindings);
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
 * Простая Laravel-like модель.
 */
abstract class Model
{
    protected static string $table = '';

    protected static string $primaryKey = 'ID';

    /**
     * Разрешённые поля для массового заполнения.
     *
     * Как $fillable в Laravel.
     */
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

3. Создай тестовую таблицу

Чтобы не трогать b_user, сделаем свою таблицу заметок.

В SQL-консоли Битрикса выполни:

CREATE TABLE IF NOT EXISTS mvc_demo_notes (
    ID INT NOT NULL AUTO_INCREMENT,
    TITLE VARCHAR(255) NOT NULL,
    BODY TEXT NULL,
    CREATED_AT DATETIME NULL,
    UPDATED_AT DATETIME NULL,
    PRIMARY KEY (ID)
);

Если у тебя PostgreSQL, скажи — дам вариант под PostgreSQL. Но раз b_user и CAST(ID AS CHAR) работают, скорее всего сейчас используется MySQL/MariaDB Битрикса.


---

4. Создай модель /local/mvc_demo/Models/Note.php

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Model;

class Note extends Model
{
    protected static string $table = 'mvc_demo_notes';

    protected static string $primaryKey = 'ID';

    protected static array $fillable = [
        'TITLE',
        'BODY',
        'CREATED_AT',
        'UPDATED_AT',
    ];

    public static function latest(int $limit = 20): array
    {
        return self::query()
            ->orderBy('ID', 'desc')
            ->limit($limit)
            ->get();
    }
}


---

5. Создай контроллер /local/mvc_demo/Controllers/NoteController.php

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Flash;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Models\Note;

class NoteController extends Controller
{
    public function index(): Response
    {
        return $this->render('notes/index', [
            'title' => 'Заметки',
            'notes' => Note::latest(20),
        ]);
    }

    public function store(): Response
    {
        $title = trim((string)request('title', ''));
        $body = trim((string)request('body', ''));

        if ($title === '') {
            Flash::error('Введите название заметки.');
            Flash::old([
                'title' => $title,
                'body' => $body,
            ]);

            return redirect()->route('notes.index');
        }

        Note::create([
            'TITLE' => $title,
            'BODY' => $body,
            'CREATED_AT' => date('Y-m-d H:i:s'),
            'UPDATED_AT' => date('Y-m-d H:i:s'),
        ]);

        Flash::success('Заметка создана.');

        return redirect()->route('notes.index');
    }

    public function delete(string $id): Response
    {
        Note::deleteById((int)$id);

        Flash::success('Заметка удалена.');

        return redirect()->route('notes.index');
    }
}


---

6. Создай view /local/mvc_demo/Views/notes/index.php

Сначала папка:

/local/mvc_demo/Views/notes/

Файл:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= e($title ?? 'Заметки') ?>
    </h1>

    <p class="mvc-page-text">
        Это тестовый CRUD через Laravel-like Model и QueryBuilder.
    </p>

    <?php if (!empty($flash)): ?>
        <?php foreach ($flash as $item): ?>
            <?php
            $type = $item['type'] ?? 'info';
            $style = 'border-color:#bfdbfe;background:#eff6ff;color:#1d4ed8;';

            if ($type === 'success') {
                $style = 'border-color:#bbf7d0;background:#f0fdf4;color:#166534;';
            } elseif ($type === 'error') {
                $style = 'border-color:#fecaca;background:#fef2f2;color:#991b1b;';
            }
            ?>

            <div class="mvc-info" style="<?= e($style) ?>">
                <?= e($item['message'] ?? '') ?>
            </div>
        <?php endforeach; ?>
    <?php endif; ?>

    <div class="mvc-info">
        <form method="post" action="<?= e(route('notes.store')) ?>">
            <?= csrf_field() ?>

            <div style="margin-bottom: 14px;">
                <label style="display:block;margin-bottom:6px;font-weight:600;">
                    Название
                </label>

                <input
                    type="text"
                    name="title"
                    value="<?= e(old('title')) ?>"
                    style="width:100%;min-height:42px;padding:8px 12px;border:1px solid #d1d5db;border-radius:10px;"
                >
            </div>

            <div style="margin-bottom: 14px;">
                <label style="display:block;margin-bottom:6px;font-weight:600;">
                    Текст
                </label>

                <textarea
                    name="body"
                    rows="4"
                    style="width:100%;padding:8px 12px;border:1px solid #d1d5db;border-radius:10px;"
                ><?= e(old('body')) ?></textarea>
            </div>

            <button
                type="submit"
                style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#2563eb;color:#fff;font-weight:600;cursor:pointer;"
            >
                Создать заметку
            </button>
        </form>
    </div>

    <div class="mvc-info">
        <?php if (empty($notes)): ?>
            <p style="margin:0;">Заметок пока нет.</p>
        <?php else: ?>
            <table style="width:100%;border-collapse:collapse;">
                <thead>
                    <tr>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">ID</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Название</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Текст</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Дата</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Действие</th>
                    </tr>
                </thead>

                <tbody>
                    <?php foreach ($notes as $note): ?>
                        <tr>
                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['ID'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['TITLE'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['BODY'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['CREATED_AT'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <form method="post" action="<?= e(route('notes.delete', ['id' => (int)$note['ID']])) ?>">
                                    <?= csrf_field() ?>
                                    <?= method_field('DELETE') ?>

                                    <button
                                        type="submit"
                                        onclick="return confirm('Удалить заметку?')"
                                        style="padding:6px 10px;border:0;border-radius:8px;background:#dc2626;color:#fff;cursor:pointer;"
                                    >
                                        Удалить
                                    </button>
                                </form>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>
</div>


---

7. Добавь маршруты в /local/mvc_demo/routes.php

Добавь use:

use Local\MvcDemo\Controllers\NoteController;

И маршруты:

Route::get('/notes', [NoteController::class, 'index'])
    ->name('notes.index');

Route::post('/notes', [NoteController::class, 'store'])
    ->middleware('csrf')
    ->name('notes.store');

Route::delete('/notes/{id:\d+}', [NoteController::class, 'delete'])
    ->middleware('csrf')
    ->name('notes.delete');


---

8. Добавь ссылку в меню

В /local/mvc_demo/Views/layouts/app.php добавь:

<a href="<?= e(route('notes.index')) ?>">Notes</a>


---

9. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Проверь:

1. Создай заметку.
2. Она должна появиться в таблице.
3. Удали заметку.
4. Она должна исчезнуть.


---

Что мы сделали

Теперь модель умеет Laravel-like CRUD:

Note::create([...]);

Note::updateById($id, [...]);

Note::deleteById($id);

И внутри это работает через QueryBuilder:

QueryBuilder::table('mvc_demo_notes')->insert([...]);
QueryBuilder::table('mvc_demo_notes')->update([...]);
QueryBuilder::table('mvc_demo_notes')->delete();

Главная мысль:

QueryBuilder — собирает SQL.
Model — даёт красивый интерфейс.
Controller — управляет действием.
View — показывает данные.

Это уже очень похоже на Laravel-подход к CRUD.