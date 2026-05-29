Отлично, тогда делаем таблицу не в public, а в схеме:

mvc

То есть таблица будет:

mvc.mvc_demo_notes

3. Создай таблицу в PostgreSQL в схеме mvc

В pgAdmin выполни:

CREATE TABLE IF NOT EXISTS mvc.mvc_demo_notes (
    id BIGSERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    body TEXT NULL,
    created_at TIMESTAMP NULL,
    updated_at TIMESTAMP NULL
);

Проверить можно так:

SELECT *
FROM mvc.mvc_demo_notes
ORDER BY id DESC;


---

4. Создай модель /local/mvc_demo/Models/Note.php

Так как PostgreSQL обычно отдаёт поля в нижнем регистре (id, title, body), сделаем модель сразу под PostgreSQL.

<?php

namespace Local\MvcDemo\Models;

use Local\Mvc\Core\Model;

class Note extends Model
{
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
        return self::query()
            ->orderBy('id', 'desc')
            ->limit($limit)
            ->get();
    }
}

Обрати внимание:

protected static string $table = 'mvc.mvc_demo_notes';

Это значит:

схема mvc
таблица mvc_demo_notes


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
            'title' => $title,
            'body' => $body,
            'created_at' => date('Y-m-d H:i:s'),
            'updated_at' => date('Y-m-d H:i:s'),
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

Создай папку:

/local/mvc_demo/Views/notes/

Создай файл:

/local/mvc_demo/Views/notes/index.php

Код:

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
        Это тестовый CRUD через Laravel-like Model и QueryBuilder. Таблица лежит в PostgreSQL:
        <span class="mvc-code">mvc.mvc_demo_notes</span>
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
                                <?= e($note['id'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['title'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['body'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?= e($note['created_at'] ?? '') ?>
                            </td>

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
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>
</div>


---

7. Добавь маршруты в /local/mvc_demo/routes.php

Вверху добавь:

use Local\MvcDemo\Controllers\NoteController;

Ниже добавь маршруты:

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

Файл:

/local/mvc_demo/Views/layouts/app.php

В меню добавь:

<a href="<?= e(route('notes.index')) ?>">Notes</a>


---

9. Проверь

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/notes

Проверь:

1. Создай заметку.
2. Она должна появиться в таблице.
3. Проверь в pgAdmin таблицу mvc.mvc_demo_notes.
4. Удали заметку.

Главное отличие от прошлого варианта: мы теперь работаем с PostgreSQL-схемой mvc, а не с таблицей в public.