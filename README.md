Проблема простая: file.* сейчас не подключены в роутер.

Поэтому:

file.upload → UNKNOWN_ACTION

files.php при загрузке страницы вызывает file.list, и из-за этого тоже падает


Что сделать

Замени /local/sitebuilder/api/index.php на этот вариант:

<?php

require_once __DIR__ . '/bootstrap.php';

$action = (string)($_POST['action'] ?? '');

if ($action === 'ping') {
    require __DIR__ . '/handlers/common.php';
    exit;
}

if (
    $action === 'site.list' ||
    $action === 'site.get' ||
    $action === 'site.create' ||
    $action === 'site.delete' ||
    $action === 'site.setHome'
) {
    require __DIR__ . '/handlers/site.php';
    exit;
}

if (
    $action === 'page.list' ||
    $action === 'page.create' ||
    $action === 'page.delete' ||
    $action === 'page.duplicate' ||
    $action === 'page.updateMeta' ||
    $action === 'page.setStatus' ||
    $action === 'page.setParent' ||
    $action === 'page.move'
) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

if (
    $action === 'menu.list' ||
    $action === 'menu.create' ||
    $action === 'menu.update' ||
    $action === 'menu.delete' ||
    $action === 'menu.setTop' ||
    $action === 'menu.item.add' ||
    $action === 'menu.item.update' ||
    $action === 'menu.item.delete' ||
    $action === 'menu.item.move'
) {
    require __DIR__ . '/handlers/menu.php';
    exit;
}

if (
    $action === 'block.list' ||
    $action === 'block.create' ||
    $action === 'block.update' ||
    $action === 'block.delete' ||
    $action === 'block.duplicate' ||
    $action === 'block.move' ||
    $action === 'block.reorder'
) {
    require __DIR__ . '/handlers/block.php';
    exit;
}

if (
    $action === 'file.list' ||
    $action === 'file.upload' ||
    $action === 'file.delete'
) {
    require __DIR__ . '/handlers/file.php';
    exit;
}

sb_json_error('UNKNOWN_ACTION', 400, [
    'action' => $action,
    'file' => __FILE__,
]);

Потом

1. Сбрось opcache


2. Снова открой files.php



Если после этого будет новая ошибка

Тогда пришли уже новый текст из debug-блока.
Скорее всего это будет уже не UNKNOWN_ACTION, а конкретный DISK_ERROR, и тогда я точечно поправлю lib/disk.php.