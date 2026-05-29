Отлично. Теперь сделаем редактирование заметки, то есть полноценный CRUD ближе к Laravel:

Create  — создать заметку
Read    — показать список
Update  — изменить заметку
Delete  — удалить заметку

Сейчас у нас есть создание и удаление. Добавим:

GET /notes/{id}/edit
PUT /notes/{id}


---

1. Обнови /local/mvc_demo/Models/Note.php

Полностью замени файл:

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

2. Создай /local/mvc_demo/Requests/UpdateNoteRequest.php

<?php

namespace Local\MvcDemo\Requests;

use Local\Mvc\Core\FormRequest;

class UpdateNoteRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'title' => ['required', 'min:2', 'max:255'],
            'body' => ['max:2000'],
        ];
    }

    public function messages(): array
    {
        return [
            'title.required' => 'Введите название заметки.',
            'title.min' => 'Название должно быть не короче 2 символов.',
            'title.max' => 'Название должно быть не длиннее 255 символов.',
            'body.max' => 'Текст заметки должен быть не длиннее 2000 символов.',
        ];
    }

    /**
     * Для редактирования оставляем null.
     * Тогда при ошибке валидации фреймворк вернёт назад по HTTP_REFERER.
     */
    public function redirectRoute(): ?string
    {
        return null;
    }
}


---

3. Обнови /local/mvc_demo/Controllers/NoteController.php

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
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
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
            'updated_at' => date('Y-m-d H:i:s'),
        ]);

        Flash::success('Заметка обновлена.');

        return redirect()->route('notes.index');
    }

    public function delete(string $id): Response
    {
        Note::deleteById((int)$id);

        Flash::success('Заметка удалена.');

        return redirect()->route('notes.index');
    }
}

Обрати внимание, стало почти как в Laravel:

public function update(string $id, UpdateNoteRequest $request): Response
{
    $data = $request->validated();
}


---

4. Создай view /local/mvc_demo/Views/notes/edit.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$noteId = (int)($note['id'] ?? 0);

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= e($title ?? 'Редактирование заметки') ?>
    </h1>

    <p class="mvc-page-text">
        Это форма редактирования. Она отправляет POST, но через
        <span class="mvc-code">method_field('PUT')</span>
        фреймворк воспринимает запрос как PUT.
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
        <form method="post" action="<?= e(route('notes.update', ['id' => $noteId])) ?>">
            <?= csrf_field() ?>
            <?= method_field('PUT') ?>

            <div style="margin-bottom: 14px;">
                <label style="display:block;margin-bottom:6px;font-weight:600;">
                    Название
                </label>

                <input
                    type="text"
                    name="title"
                    value="<?= e(old('title', $note['title'] ?? '')) ?>"
                    style="width:100%;min-height:42px;padding:8px 12px;border:1px solid #d1d5db;border-radius:10px;"
                >
            </div>

            <div style="margin-bottom: 14px;">
                <label style="display:block;margin-bottom:6px;font-weight:600;">
                    Текст
                </label>

                <textarea
                    name="body"
                    rows="5"
                    style="width:100%;padding:8px 12px;border:1px solid #d1d5db;border-radius:10px;"
                ><?= e(old('body', $note['body'] ?? '')) ?></textarea>
            </div>

            <div style="display:flex;gap:10px;align-items:center;">
                <button
                    type="submit"
                    style="min-height:42px;padding:0 18px;border:0;border-radius:10px;background:#2563eb;color:#fff;font-weight:600;cursor:pointer;"
                >
                    Сохранить
                </button>

                <a href="<?= e(route('notes.index')) ?>">
                    ← Назад к списку
                </a>
            </div>
        </form>
    </div>
</div>


---

5. Обнови /local/mvc_demo/Views/notes/index.php

В таблице в колонке Действие сейчас есть только форма удаления.

Найди этот кусок:

<td style="padding:8px;border-bottom:1px solid #e5e7eb;">
    <form method="post" action="<?= e(route('notes.delete', ['id' => (int)($note['id'] ?? 0)])) ?>">
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

Замени на:

<td style="padding:8px;border-bottom:1px solid #e5e7eb;">
    <div style="display:flex;gap:8px;align-items:center;">
        <a
            href="<?= e(route('notes.edit', ['id' => (int)($note['id'] ?? 0)])) ?>"
            style="padding:6px 10px;border-radius:8px;background:#2563eb;color:#fff;text-decoration:none;"
        >
            Изменить
        </a>

        <form method="post" action="<?= e(route('notes.delete', ['id' => (int)($note['id'] ?? 0)])) ?>" style="margin:0;">
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
    </div>
</td>


---

6. Обнови маршруты /local/mvc_demo/routes.php

У тебя уже есть:

Route::get('/notes', [NoteController::class, 'index'])
    ->name('notes.index');

Route::post('/notes', [NoteController::class, 'store'])
    ->middleware('csrf')
    ->name('notes.store');

Route::delete('/notes/{id:\d+}', [NoteController::class, 'delete'])
    ->middleware('csrf')
    ->name('notes.delete');

Добавь между store и delete:

Route::get('/notes/{id:\d+}/edit', [NoteController::class, 'edit'])
    ->name('notes.edit');

Route::put('/notes/{id:\d+}', [NoteController::class, 'update'])
    ->middleware('csrf')
    ->name('notes.update');

Итоговый блок:

Route::get('/notes', [NoteController::class, 'index'])
    ->name('notes.index');

Route::post('/notes', [NoteController::class, 'store'])
    ->middleware('csrf')
    ->name('notes.store');

Route::get('/notes/{id:\d+}/edit', [NoteController::class, 'edit'])
    ->name('notes.edit');

Route::put('/notes/{id:\d+}', [NoteController::class, 'update'])
    ->middleware('csrf')
    ->name('notes.update');

Route::delete('/notes/{id:\d+}', [NoteController::class, 'delete'])
    ->middleware('csrf')
    ->name('notes.delete');


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Проверь:

1. Нажми "Изменить".
2. Измени название или текст.
3. Нажми "Сохранить".
4. Должно вернуть на список.
5. В таблице должна быть обновлённая заметка.
6. В pgAdmin в ProjectsDB → mvc → mvc_demo_notes тоже должно обновиться.


---

Что мы сделали

Теперь у нас Laravel-like CRUD:

Route::get('/notes', [NoteController::class, 'index'])->name('notes.index');
Route::post('/notes', [NoteController::class, 'store'])->name('notes.store');
Route::get('/notes/{id}/edit', [NoteController::class, 'edit'])->name('notes.edit');
Route::put('/notes/{id}', [NoteController::class, 'update'])->name('notes.update');
Route::delete('/notes/{id}', [NoteController::class, 'delete'])->name('notes.delete');

И контроллер похож на Laravel:

public function store(StoreNoteRequest $request): Response
public function update(string $id, UpdateNoteRequest $request): Response

Это уже очень хорошая учебная база для понимания Laravel resource controllers.