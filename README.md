Отлично, это даже безопаснее. api.php пока не трогаем, а продолжаем готовить фундамент вокруг него.

Тогда следующий правильный шаг — добавить ещё 3 файла в lib:

menu.php

layout.php

disk.php


И только потом уже менять api.php целиком.


---

1. lib/menu.php

Создай файл lib/menu.php и вставь:

<?php
declare(strict_types=1);

function sb_menu_get_site_record(array $all, int $siteId): ?array
{
    foreach ($all as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) {
            return $r;
        }
    }
    return null;
}

function sb_menu_upsert_site_record(array &$all, int $siteId, array $record): void
{
    $found = false;

    foreach ($all as $i => $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) {
            $all[$i] = $record;
            $found = true;
            break;
        }
    }

    if (!$found) {
        $all[] = $record;
    }
}

function sb_menu_next_menu_id(array $siteRecord): int
{
    $max = 0;
    foreach (($siteRecord['menus'] ?? []) as $m) {
        $max = max($max, (int)($m['id'] ?? 0));
    }
    return $max + 1;
}

function sb_menu_next_item_id(array $menu): int
{
    $max = 0;
    foreach (($menu['items'] ?? []) as $it) {
        $max = max($max, (int)($it['id'] ?? 0));
    }
    return $max + 1;
}

function sb_menu_next_sort(array $items): int
{
    $max = 0;
    foreach ($items as $it) {
        $max = max($max, (int)($it['sort'] ?? 0));
    }
    return $max + 10;
}

function sb_menu_find_menu(array $siteRecord, int $menuId): ?array
{
    foreach (($siteRecord['menus'] ?? []) as $m) {
        if ((int)($m['id'] ?? 0) === $menuId) {
            return $m;
        }
    }
    return null;
}

function sb_menu_update_menu(array &$siteRecord, int $menuId, callable $fn): bool
{
    $menus = $siteRecord['menus'] ?? [];
    $changed = false;

    foreach ($menus as $i => $m) {
        if ((int)($m['id'] ?? 0) === $menuId) {
            $menus[$i] = $fn($m);
            $changed = true;
            break;
        }
    }

    if ($changed) {
        $siteRecord['menus'] = $menus;
    }

    return $changed;
}


---

2. lib/layout.php

Создай файл lib/layout.php и вставь:

<?php
declare(strict_types=1);

function sb_layout_empty_record(int $siteId): array
{
    return [
        'siteId' => $siteId,
        'zones' => [
            'header' => [],
            'footer' => [],
            'left'   => [],
            'right'  => [],
        ],
    ];
}

function sb_layout_find_record(array $all, int $siteId): ?array
{
    foreach ($all as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) {
            return $r;
        }
    }
    return null;
}

function sb_layout_upsert_record(array &$all, int $siteId, array $record): void
{
    $found = false;

    foreach ($all as $i => $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) {
            $all[$i] = $record;
            $found = true;
            break;
        }
    }

    if (!$found) {
        $all[] = $record;
    }
}

function sb_layout_ensure_record(int $siteId): array
{
    $all = sb_read_layouts();
    $rec = sb_layout_find_record($all, $siteId);

    if ($rec) {
        return $rec;
    }

    $rec = sb_layout_empty_record($siteId);
    $all[] = $rec;
    sb_write_layouts($all);

    return $rec;
}

function sb_layout_valid_zone(string $zone): bool
{
    return in_array($zone, ['header', 'footer', 'left', 'right'], true);
}

function sb_layout_zone_blocks(array $record, string $zone): array
{
    $zones = $record['zones'] ?? [];
    $blocks = $zones[$zone] ?? [];
    return is_array($blocks) ? $blocks : [];
}

function sb_layout_zone_set(array &$record, string $zone, array $blocks): void
{
    if (!isset($record['zones']) || !is_array($record['zones'])) {
        $record['zones'] = [];
    }

    $record['zones'][$zone] = array_values($blocks);
}

function sb_layout_next_block_id(array $record): int
{
    $max = 0;
    $zones = is_array($record['zones'] ?? null) ? $record['zones'] : [];

    foreach ($zones as $zoneBlocks) {
        if (!is_array($zoneBlocks)) {
            continue;
        }

        foreach ($zoneBlocks as $b) {
            $max = max($max, (int)($b['id'] ?? 0));
        }
    }

    return $max + 1;
}

function sb_layout_next_sort(array $blocks): int
{
    $max = 0;
    foreach ($blocks as $b) {
        $max = max($max, (int)($b['sort'] ?? 0));
    }
    return $max + 10;
}

function sb_layout_find_block(array $record, string $zone, int $blockId): ?array
{
    $blocks = sb_layout_zone_blocks($record, $zone);

    foreach ($blocks as $b) {
        if ((int)($b['id'] ?? 0) === $blockId) {
            return $b;
        }
    }

    return null;
}


---

3. lib/disk.php

Создай файл lib/disk.php и вставь:

<?php
declare(strict_types=1);

function sb_disk_add_subfolder(Folder $parent, array $fields, Storage $storage): Folder
{
    try {
        $folder = $parent->addSubFolder($fields, []);
        if ($folder instanceof Folder) {
            return $folder;
        }
    } catch (\Throwable $e) {
    }

    $ctx = $storage->getSecurityContext($GLOBALS['USER']);
    $folder = $parent->addSubFolder($fields, $ctx);

    if ($folder instanceof Folder) {
        return $folder;
    }

    throw new RuntimeException('CANNOT_CREATE_FOLDER');
}

function sb_disk_upload_file(Folder $folder, array $fileArray, array $fields, Storage $storage): File
{
    try {
        $obj = $folder->uploadFile($fileArray, $fields, []);
        if ($obj instanceof File) {
            return $obj;
        }
    } catch (\Throwable $e) {
    }

    $ctx = $storage->getSecurityContext($GLOBALS['USER']);
    $obj = $folder->uploadFile($fileArray, $fields, $ctx);

    if ($obj instanceof File) {
        return $obj;
    }

    throw new RuntimeException('UPLOAD_FAILED');
}

function sb_disk_get_children(Folder $folder, Storage $storage): array
{
    try {
        $ctx = $storage->getSecurityContext($GLOBALS['USER']);
        return $folder->getChildren($ctx);
    } catch (\Throwable $e) {
    }

    return $folder->getChildren();
}

function sb_disk_common_storage(): Storage
{
    if (!Loader::includeModule('disk')) {
        throw new RuntimeException('DISK_NOT_INSTALLED');
    }

    if (method_exists(\Bitrix\Disk\Storage::class, 'loadByEntity')) {
        $storage = \Bitrix\Disk\Storage::loadByEntity('common', 0);
        if ($storage) {
            return $storage;
        }
    }

    $driver = \Bitrix\Disk\Driver::getInstance();
    if (method_exists($driver, 'getStorageByCommonId')) {
        $storage = $driver->getStorageByCommonId('shared_files_' . SITE_ID);
        if ($storage) {
            return $storage;
        }
    }

    throw new RuntimeException('COMMON_STORAGE_NOT_FOUND');
}

function sb_disk_get_or_create_root(Storage $storage, string $name): Folder
{
    $root = $storage->getRootObject();

    $child = $root->getChild([
        '=NAME' => $name,
        '=TYPE' => \Bitrix\Disk\Internals\ObjectTable::TYPE_FOLDER,
    ]);

    if ($child instanceof Folder) {
        return $child;
    }

    return sb_disk_add_subfolder($root, [
        'NAME' => $name,
        'CREATED_BY' => (int)$GLOBALS['USER']->GetID(),
    ], $storage);
}

function sb_disk_ensure_site_folder(int $siteId): Folder
{
    $sites = sb_read_sites();
    $site = null;
    $siteIndex = null;

    foreach ($sites as $i => $s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $site = $s;
            $siteIndex = $i;
            break;
        }
    }

    if (!$site) {
        throw new RuntimeException('SITE_NOT_FOUND');
    }

    $storage = sb_disk_common_storage();

    $diskFolderId = (int)($site['diskFolderId'] ?? 0);
    if ($diskFolderId > 0) {
        $folder = Folder::loadById($diskFolderId);
        if ($folder) {
            return $folder;
        }
    }

    $root = sb_disk_get_or_create_root($storage, 'SiteBuilder');

    $slug = (string)($site['slug'] ?? ('site-' . $siteId));
    $slug = $slug !== '' ? $slug : ('site-' . $siteId);

    $existing = $root->getChild([
        '=NAME' => $slug,
        '=TYPE' => \Bitrix\Disk\Internals\ObjectTable::TYPE_FOLDER,
    ]);

    if ($existing instanceof Folder) {
        $folder = $existing;
    } else {
        $folder = sb_disk_add_subfolder($root, [
            'NAME' => $slug,
            'CREATED_BY' => (int)$GLOBALS['USER']->GetID(),
        ], $storage);
    }

    $sites[$siteIndex]['diskFolderId'] = (int)$folder->getId();
    sb_write_sites($sites);

    return $folder;
}

function sb_disk_sync_folder_rights(int $siteId, Folder $folder): void
{
    $rm = Driver::getInstance()->getRightsManager();

    if (!method_exists($rm, 'setRights')) {
        return;
    }

    $taskRead = (method_exists($rm, 'getTaskIdByName') && defined(get_class($rm) . '::TASK_READ'))
        ? $rm->getTaskIdByName($rm::TASK_READ) : null;
    $taskEdit = (method_exists($rm, 'getTaskIdByName') && defined(get_class($rm) . '::TASK_EDIT'))
        ? $rm->getTaskIdByName($rm::TASK_EDIT) : null;
    $taskFull = (method_exists($rm, 'getTaskIdByName') && defined(get_class($rm) . '::TASK_FULL'))
        ? $rm->getTaskIdByName($rm::TASK_FULL) : null;

    if (!$taskRead || !$taskEdit || !$taskFull) {
        return;
    }

    $acc = sb_read_access();
    $acc = array_values(array_filter($acc, fn($r) => (int)($r['siteId'] ?? 0) === $siteId));

    $rights = [];
    foreach ($acc as $r) {
        $code = (string)($r['accessCode'] ?? '');
        if ($code === '') {
            continue;
        }

        $role = strtoupper((string)($r['role'] ?? 'VIEWER'));
        $taskId = $taskRead;

        if ($role === 'EDITOR') {
            $taskId = $taskEdit;
        }
        if ($role === 'ADMIN' || $role === 'OWNER') {
            $taskId = $taskFull;
        }

        $rights[] = [
            'ACCESS_CODE' => $code,
            'TASK_ID' => $taskId,
        ];
    }

    $rm->setRights($folder, $rights);
}

function sb_disk_file_belongs_to_site(int $siteId, int $fileId): bool
{
    if ($fileId <= 0) {
        return false;
    }

    $folder = sb_disk_ensure_site_folder($siteId);
    $file = \Bitrix\Disk\File::loadById($fileId);

    if (!$file) {
        return false;
    }

    return ((int)$file->getParentId() === (int)$folder->getId());
}


---

Что делать сейчас

Пока api.php не трогаем, как ты и сказал.

Сейчас просто:

1. Создай эти 3 файла.


2. Вставь туда код.


3. Ничего ещё не подключай.




---

Следующий шаг

После этого пойдём к следующему безопасному этапу:

соберём новый api.php-каркас, который будет:

подключать все lib/*

но пока ещё не резать actions на отдельные файлы


Это будет промежуточный безопасный шаг перед полным рефакторингом api.php.

Когда добавишь эти 3 файла, напиши, и я дам следующий шаг.