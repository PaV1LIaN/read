Идём дальше. Раз мы сделали Policy, следующий Laravel-like шаг — сделать helper can() / cannot() для views.

В Laravel часто делают так:

@can('update', $note)
    <a>Изменить</a>
@endcan

У нас Blade нет, поэтому сделаем PHP-вариант:

<?php if (can('update', \Local\MvcDemo\Models\Note::class)): ?>
    <a>Изменить</a>
<?php endif; ?>

То есть view сможет сам аккуратно скрывать кнопки, если прав нет.


---

1. Обнови /local/mvc/helpers.php

В конец файла добавь:

if (!function_exists('can')) {
    /**
     * Laravel-like can()
     *
     * Пример:
     * can('create', Note::class)
     * can('update', $note)
     */
    function can(string $ability, object|string $subject): bool
    {
        try {
            return \Local\Mvc\Support\Facades\Gate::allows($ability, $subject);
        } catch (\Throwable $e) {
            return false;
        }
    }
}

if (!function_exists('cannot')) {
    /**
     * Laravel-like cannot()
     *
     * Пример:
     * cannot('delete', $note)
     */
    function cannot(string $ability, object|string $subject): bool
    {
        return !can($ability, $subject);
    }
}

Теперь в любом view можно писать:

can('create', \Local\MvcDemo\Models\Note::class)


---

2. Обнови /local/mvc_demo/Policies/NotePolicy.php

Сделаем методы update и delete чуть гибче: они смогут принимать и объект Note, и строку Note::class.

Полностью замени файл:

<?php

namespace Local\MvcDemo\Policies;

use Local\Mvc\Core\Auth;
use Local\MvcDemo\Models\Note;

/**
 * NotePolicy
 *
 * Правила доступа к заметкам.
 *
 * Для demo:
 * - смотреть список может авторизованный пользователь
 * - создавать, редактировать и удалять может только админ
 */
class NotePolicy
{
    public function viewAny(string $modelClass): bool
    {
        return Auth::check();
    }

    public function create(string $modelClass): bool
    {
        return Auth::isAdmin();
    }

    public function update(Note|string $note): bool
    {
        return Auth::isAdmin();
    }

    public function delete(Note|string $note): bool
    {
        return Auth::isAdmin();
    }
}

Почему так?

В контроллере мы проверяем конкретную модель:

$this->authorize('update', $note);

А во view иногда удобно проверить просто класс:

can('update', Note::class)


---

3. Обнови /local/mvc_demo/Views/notes/index.php

В самом верху после проверки B_PROLOG_INCLUDED добавь:

$canCreateNote = can('create', \Local\MvcDemo\Models\Note::class);
$canUpdateNote = can('update', \Local\MvcDemo\Models\Note::class);
$canDeleteNote = can('delete', \Local\MvcDemo\Models\Note::class);

Должно быть примерно так:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$canCreateNote = can('create', \Local\MvcDemo\Models\Note::class);
$canUpdateNote = can('update', \Local\MvcDemo\Models\Note::class);
$canDeleteNote = can('delete', \Local\MvcDemo\Models\Note::class);

?>


---

4. Спрячь форму создания от тех, кто не может создавать

Найди блок с формой создания заметки:

<div class="mvc-info">
    <form method="post" action="<?= e(route('notes.store')) ?>">
        ...
    </form>
</div>

Оберни его так:

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
<?php else: ?>
    <div class="mvc-info" style="border-color:#fde68a;background:#fffbeb;color:#92400e;">
        У вас нет прав на создание заметок.
    </div>
<?php endif; ?>


---

5. Спрячь кнопку Factory

Если ты добавлял блок:

<form method="post" action="<?= e(route('notes.factory')) ?>">

оберни его так:

<?php if ($canCreateNote): ?>
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
<?php endif; ?>


---

6. Обнови колонку действий

В таблице найди колонку с кнопками Изменить и Удалить.

Сделай так:

<td style="padding:8px;border-bottom:1px solid #e5e7eb;">
    <?php if ($canUpdateNote || $canDeleteNote): ?>
        <div style="display:flex;gap:8px;align-items:center;">
            <?php if ($canUpdateNote): ?>
                <a
                    href="<?= e(route('notes.edit', ['id' => (int)($note['id'] ?? 0)])) ?>"
                    style="padding:6px 10px;border-radius:8px;background:#2563eb;color:#fff;text-decoration:none;"
                >
                    Изменить
                </a>
            <?php endif; ?>

            <?php if ($canDeleteNote): ?>
                <form method="post" action="<?= e(route('notes.destroy', ['id' => (int)($note['id'] ?? 0)])) ?>" style="margin:0;">
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
            <?php endif; ?>
        </div>
    <?php else: ?>
        <span style="color:#9ca3af;">Нет действий</span>
    <?php endif; ?>
</td>


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/notes

Если ты админ — кнопки создания, изменения, удаления должны быть видны.

Если зайти обычным пользователем:

Создать заметку — скрыто
Factory — скрыто
Изменить / удалить — скрыто


---

Что мы сделали

Раньше права проверялись только в контроллере:

$this->authorize('update', $note);

Теперь view тоже может аккуратно проверять права:

can('update', Note::class)

Главная мысль:

Policy защищает действие.
can() помогает не показывать лишние кнопки.

Это похоже на Laravel:

@can('update', $note)
    ...
@endcan

Только у нас пока PHP-вариант:

<?php if (can('update', Note::class)): ?>
    ...
<?php endif; ?>