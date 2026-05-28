Да, вижу проблему на скрине.

У тебя после отправки зелёное сообщение работает, но текст успеха попал в поле “Сообщение”.

Причина простая: в form/index.php мы использовали переменную $message и для поля формы, и для flash-сообщения. PHP перезаписал её.

Нужно заменить файл формы.

Замени полностью /local/mvc_demo/Views/form/index.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

$formName = $old['name'] ?? '';
$formMessage = $old['message'] ?? '';

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Форма') ?>
    </h1>

    <p class="mvc-page-text">
        Это простая форма, чтобы проверить POST-запросы в нашем MVC.
    </p>

    <?php if (!empty($flash)): ?>
        <?php foreach ($flash as $item): ?>
            <?php
            $flashType = $item['type'] ?? 'info';
            $flashMessage = $item['message'] ?? '';

            $style = 'border-color: #bfdbfe; background: #eff6ff; color: #1d4ed8;';

            if ($flashType === 'success') {
                $style = 'border-color: #bbf7d0; background: #f0fdf4; color: #166534;';
            } elseif ($flashType === 'error') {
                $style = 'border-color: #fecaca; background: #fef2f2; color: #991b1b;';
            } elseif ($flashType === 'warning') {
                $style = 'border-color: #fde68a; background: #fffbeb; color: #92400e;';
            }
            ?>

            <div class="mvc-info" style="<?= htmlspecialcharsbx($style) ?>">
                <?= htmlspecialcharsbx($flashMessage) ?>
            </div>
        <?php endforeach; ?>
    <?php endif; ?>

    <?php if (!empty($errors)): ?>
        <div class="mvc-info" style="border-color: #fecaca; background: #fef2f2;">
            <b style="color: #991b1b;">Ошибки:</b>

            <ol>
                <?php foreach ($errors as $error): ?>
                    <li style="color: #991b1b;">
                        <?= htmlspecialcharsbx($error) ?>
                    </li>
                <?php endforeach; ?>
            </ol>
        </div>
    <?php endif; ?>

    <?php if (!empty($success)): ?>
        <div class="mvc-info" style="border-color: #bbf7d0; background: #f0fdf4;">
            <b style="color: #166534;">Успешно:</b>

            <p style="color: #166534; margin-bottom: 0;">
                <?= htmlspecialcharsbx($success) ?>
            </p>
        </div>
    <?php endif; ?>

    <form method="post" action="/local/mvc_demo/form/send" style="margin-top: 24px;">
        <?php if (function_exists('bitrix_sessid_post')): ?>
            <?= bitrix_sessid_post() ?>
        <?php endif; ?>

        <div style="margin-bottom: 16px;">
            <label style="display: block; margin-bottom: 6px; font-weight: 600;">
                Имя
            </label>

            <input
                type="text"
                name="name"
                value="<?= htmlspecialcharsbx($formName) ?>"
                style="width: 100%; min-height: 42px; padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 10px;"
            >
        </div>

        <div style="margin-bottom: 16px;">
            <label style="display: block; margin-bottom: 6px; font-weight: 600;">
                Сообщение
            </label>

            <textarea
                name="message"
                rows="5"
                style="width: 100%; padding: 8px 12px; border: 1px solid #d1d5db; border-radius: 10px;"
            ><?= htmlspecialcharsbx($formMessage) ?></textarea>
        </div>

        <button
            type="submit"
            style="min-height: 42px; padding: 0 18px; border: 0; border-radius: 10px; background: #2563eb; color: #fff; font-weight: 600; cursor: pointer;"
        >
            Отправить
        </button>
    </form>
</div>

Что исправили

Было плохо:

$message = $old['message'] ?? '';

И потом ниже flash тоже использовал:

$message = $item['message'] ?? '';

То есть flash перезаписывал сообщение формы.

Теперь отдельно:

$formMessage

и отдельно:

$flashMessage

После замены сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Потом отправь форму ещё раз. Зелёное сообщение должно остаться сверху, а поле “Сообщение” после успешной отправки должно быть пустым.