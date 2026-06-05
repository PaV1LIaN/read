Да, замени полностью эти два файла.

/local/mvc_demo/Views/notes/index.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$canCreateNote = can('create', \Local\MvcDemo\Models\Note::class);
$canUpdateNote = can('update', \Local\MvcDemo\Models\Note::class);
$canDeleteNote = can('delete', \Local\MvcDemo\Models\Note::class);

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= e($title ?? 'Заметки') ?>
    </h1>

    <p class="mvc-page-text">
        Это тестовый CRUD через Laravel-like Model, QueryBuilder, FormRequest, Policy, Factory, Soft Deletes и Pagination.
        Таблица лежит в PostgreSQL:
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
        <a href="<?= e(route('notes.trash')) ?>">
            Открыть корзину удалённых заметок
        </a>
    </div>

    <?php if ($canCreateNote): ?>
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
    <?php else: ?>
        <div class="mvc-info" style="border-color:#fde68a;background:#fffbeb;color:#92400e;">
            У вас нет прав на создание заметок.
        </div>
    <?php endif; ?>

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
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Создана</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Обновлена</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Действие</th>
                    </tr>
                </thead>

                <tbody>
                    <?php foreach ($notes as $note): ?>
                        <?php
                        $noteId = (int)($note['id'] ?? 0);
                        ?>

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
                                <?= e($note['updated_at'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?php if ($canUpdateNote || $canDeleteNote): ?>
                                    <div style="display:flex;gap:8px;align-items:center;">
                                        <?php if ($canUpdateNote): ?>
                                            <a
                                                href="<?= e(route('notes.edit', ['note' => $noteId])) ?>"
                                                style="padding:6px 10px;border-radius:8px;background:#2563eb;color:#fff;text-decoration:none;"
                                            >
                                                Изменить
                                            </a>
                                        <?php endif; ?>

                                        <?php if ($canDeleteNote): ?>
                                            <form method="post" action="<?= e(route('notes.destroy', ['note' => $noteId])) ?>" style="margin:0;">
                                                <?= csrf_field() ?>
                                                <?= method_field('DELETE') ?>

                                                <button
                                                    type="submit"
                                                    onclick="return confirm('Удалить заметку? Она попадёт в корзину.')"
                                                    style="padding:6px 10px;border:0;border-radius:8px;background:#dc2626;color:#fff;cursor:pointer;"
                                                >
                                                    Удалить
                                                </button>
                                            </form>
                                        <?php endif; ?>
                                    </div>
                                <?php else: ?>
                                    <span style="color:#9ca3af;">Нет действий</span>
                                <?php endif; ?>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>

    <?php
    $routeName = 'notes.index';
    $routeParams = [];
    $query = [];
    require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
    ?>
</div>


---

/local/mvc_demo/Views/notes/trash.php

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
        <a href="<?= e(route('notes.index')) ?>">
            ← Назад к заметкам
        </a>
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
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Текст</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Удалена</th>
                        <th style="text-align:left;padding:8px;border-bottom:1px solid #e5e7eb;">Действие</th>
                    </tr>
                </thead>

                <tbody>
                    <?php foreach ($notes as $note): ?>
                        <?php
                        $noteId = (int)($note['id'] ?? 0);
                        ?>

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
                                <?= e($note['deleted_at'] ?? '') ?>
                            </td>

                            <td style="padding:8px;border-bottom:1px solid #e5e7eb;">
                                <?php if ($canRestoreNote || $canForceDeleteNote): ?>
                                    <div style="display:flex;gap:8px;align-items:center;">
                                        <?php if ($canRestoreNote): ?>
                                            <form method="post" action="<?= e(route('notes.restore', ['note' => $noteId])) ?>" style="margin:0;">
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
                                            <form method="post" action="<?= e(route('notes.force-delete', ['note' => $noteId])) ?>" style="margin:0;">
                                                <?= csrf_field() ?>
                                                <?= method_field('DELETE') ?>

                                                <button
                                                    type="submit"
                                                    onclick="return confirm('Удалить окончательно? Это действие нельзя отменить.')"
                                                    style="padding:6px 10px;border:0;border-radius:8px;background:#dc2626;color:#fff;cursor:pointer;"
                                                >
                                                    Удалить навсегда
                                                </button>
                                            </form>
                                        <?php endif; ?>
                                    </div>
                                <?php else: ?>
                                    <span style="color:#9ca3af;">Нет действий</span>
                                <?php endif; ?>
                            </td>
                        </tr>
                    <?php endforeach; ?>
                </tbody>
            </table>
        <?php endif; ?>
    </div>

    <?php
    $routeName = 'notes.trash';
    $routeParams = [];
    $query = [];
    require $_SERVER['DOCUMENT_ROOT'] . '/local/mvc_demo/Views/partials/pagination.php';
    ?>
</div>

После замены сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';