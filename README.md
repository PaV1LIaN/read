Да, по HTML кнопка собрана нормально.
Проблема почти наверняка в том, что на твоей кастомной публичной странице не выводятся head-ассеты Bitrix, поэтому расширение viewer фактически не подключается, даже если ты вызываешь CJSCore::Init(['ui.viewer']).

В обычном Bitrix-шаблоне это делает:

$APPLICATION->ShowHead();

А у тебя public_page.php — свой layout.

Что исправить

Файл

/local/sitebuilder/views/layout/public_page.php

1. В <head> добавь:

<?php
global $APPLICATION;
?>

если его еще нет.

2. Внутри <head> после <meta charset="UTF-8"> вставь:

<?php $APPLICATION->ShowHead(); ?>

То есть начало должно стать примерно таким:

<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <?php
    global $APPLICATION;
    $APPLICATION->ShowHead();
    ?>
    <title><?= sb_public_h((string)($currentPage['title'] ?? $site['name'] ?? 'SiteBuilder')) ?></title>
    <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css">

    <?php if ($pageHasDiskBlock): ?>
        <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css">
    <?php endif; ?>


---

3. Оставь внизу страницы это:

<?php if ($pageHasDiskBlock): ?>
<?php CJSCore::Init(['ui.viewer']); ?>
<script src="/bitrix/js/main/core/core.js"></script>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js"></script>
<?php endif; ?>

Но лучше CJSCore::Init(['ui.viewer']); перенести выше, до ShowHead().

Правильнее так:

В начале файла, до <!doctype html>, добавь:

<?php
global $APPLICATION;

if ($pageHasDiskBlock) {
    CJSCore::Init(['ui.viewer']);
}
?>

И тогда внизу оставь только:

<?php if ($pageHasDiskBlock): ?>
<script src="/bitrix/js/main/core/core.js"></script>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js"></script>
<?php endif; ?>


---

Почему это важно

CJSCore::Init(['ui.viewer']) не всегда просто печатает <script> сразу.
Оно регистрирует ассеты, а выводятся они через ShowHead().

Без этого у тебя:

HTML-кнопка есть

data-viewer есть

но нужного JS viewer-а на странице реально нет


И поэтому при клике ничего не происходит.


---

Что сделать после правки

1. сохранить public_page.php


2. сделать Ctrl + F5


3. в консоли проверить:



typeof BX !== 'undefined' && BX.UI && BX.UI.Viewer ? 'viewer ok' : 'viewer missing'

Если все подключилось, должно быть:

viewer ok

И еще можно проверить:

typeof BX !== 'undefined' && BX.Disk && BX.Disk.Viewer ? 'disk viewer ok' : 'disk viewer missing'


---

Если после ShowHead() docx все равно не откроется, пришли результат этих двух проверок из консоли.