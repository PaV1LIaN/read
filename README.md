Да, ты прав. settings.php — это настройки сайта, значит его должны открывать только:

ADMIN / OWNER / администратор Битрикс24

EDITOR и VIEWER не должны туда попадать.

В твоём settings.php сейчас вообще нет проверки роли, поэтому страница открывается всем авторизованным пользователям, а уже API потом может отказать. Нужно закрыть сам файл.

Исправление для /local/sitebuilder/settings.php

Вверху файла найди:

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$siteId = (int)($_GET['siteId'] ?? 0);

Сразу после этого добавь:

require_once __DIR__ . '/lib/json.php';
require_once __DIR__ . '/lib/response.php';
require_once __DIR__ . '/lib/access.php';
require_once __DIR__ . '/lib/helpers.php';

Дальше ниже у тебя есть блок:

if ($siteId <= 0) {
    ?>
    <!doctype html>
    <html lang="ru">
    ...
    </html>
    <?php
    exit;
}
?>

Сразу после этого блока, но перед ?>, добавь проверку:

if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}

В итоге начало файла должно быть таким:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

global $APPLICATION, $USER;

if (!$USER->IsAuthorized()) {
    require $_SERVER['DOCUMENT_ROOT'] . '/auth.php';
    exit;
}

CJSCore::Init(['ajax']);

header('Content-Type: text/html; charset=UTF-8');

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$siteId = (int)($_GET['siteId'] ?? 0);

require_once __DIR__ . '/lib/json.php';
require_once __DIR__ . '/lib/response.php';
require_once __DIR__ . '/lib/access.php';
require_once __DIR__ . '/lib/helpers.php';

if ($siteId <= 0) {
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>SiteBuilder / Settings</title>
        <?php $APPLICATION->ShowHead(); ?>
        <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    </head>
    <body class="sb-admin-body">
        <div class="sb-page">
            <h1 class="sb-title">Не передан siteId</h1>
            <p><a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">Вернуться к списку сайтов</a></p>
        </div>
    </body>
    </html>
    <?php
    exit;
}

if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}
?>

После этого:

VIEWER → ACCESS_DENIED
EDITOR → ACCESS_DENIED
ADMIN → откроет
OWNER → откроет
админ Битрикс24 → откроет

И ещё проверь ссылки на settings.php, если они где-то есть в интерфейсе. Их тоже нужно показывать только при roleRank >= 3.