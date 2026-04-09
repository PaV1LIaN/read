Отлично. Делаем file / Disk-пакет.

Это самый чувствительный участок, поэтому делаем практично и с совместимостью:

file.list

file.upload

file.delete


И выносим Disk-обвязку в отдельный файл:

lib/disk.php


Подход такой:

у каждого сайта есть папка в общем Disk-хранилище

diskFolderId хранится в sites.json

если папки ещё нет — создаём

список файлов отдаём из этой папки

загрузка идёт в эту папку

удаление разрешаем только файлов из папки этого сайта



---

1. Создай новый файл /local/sitebuilder/lib/disk.php

<?php

require_once __DIR__ . '/json.php';
require_once __DIR__ . '/helpers.php';

use Bitrix\Main\Loader;
use Bitrix\Disk\Driver;
use Bitrix\Disk\Storage;
use Bitrix\Disk\Folder;
use Bitrix\Disk\File;
use Bitrix\Disk\Security\FakeSecurityContext;

if (!function_exists('sb_disk_require_module')) {
    function sb_disk_require_module(): void
    {
        if (!Loader::includeModule('disk')) {
            throw new RuntimeException('Disk module is not installed');
        }
    }
}

if (!function_exists('sb_disk_security_context')) {
    function sb_disk_security_context()
    {
        global $USER;

        if (class_exists(FakeSecurityContext::class)) {
            return new FakeSecurityContext((int)$USER->GetID());
        }

        return null;
    }
}

if (!function_exists('sb_disk_common_storage')) {
    function sb_disk_common_storage(): ?Storage
    {
        sb_disk_require_module();

        if (method_exists(Storage::class, 'loadByEntity')) {
            $storage = Storage::loadByEntity('common', 0);
            if ($storage) {
                return $storage;
            }
        }

        if (class_exists(Driver::class)) {
            $driver = Driver::getInstance();

            if (method_exists($driver, 'getStorageByCommonId')) {
                $storage = $driver->getStorageByCommonId('shared_files_' . SITE_ID);
                if ($storage) {
                    return $storage;
                }
            }
        }

        return null;
    }
}

if (!function_exists('sb_disk_root_folder')) {
    function sb_disk_root_folder(): Folder
    {
        $storage = sb_disk_common_storage();
        if (!$storage) {
            throw new RuntimeException('Common Disk storage not found');
        }

        $root = $storage->getRootObject();
        if (!$root instanceof Folder) {
            throw new RuntimeException('Common Disk root folder not found');
        }

        return $root;
    }
}

if (!function_exists('sb_disk_get_children')) {
    function sb_disk_get_children(Folder $folder): array
    {
        $securityContext = sb_disk_security_context();

        try {
            if ($securityContext) {
                return (array)$folder->getChildren($securityContext);
            }
        } catch (Throwable $e) {
        }

        try {
            return (array)$folder->getChildren();
        } catch (Throwable $e) {
            return [];
        }
    }
}

if (!function_exists('sb_disk_find_child_folder_by_name')) {
    function sb_disk_find_child_folder_by_name(Folder $parent, string $name): ?Folder
    {
        foreach (sb_disk_get_children($parent) as $child) {
            if ($child instanceof Folder && (string)$child->getName() === $name) {
                return $child;
            }
        }
        return null;
    }
}

if (!function_exists('sb_disk_add_subfolder')) {
    function sb_disk_add_subfolder(Folder $parent, string $name): Folder
    {
        $fields = ['NAME' => $name];
        $securityContext = sb_disk_security_context();

        try {
            $created = $parent->addSubFolder($fields, []);
            if ($created instanceof Folder) {
                return $created;
            }
            if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof Folder) {
                return $created['OBJECT'];
            }
        } catch (Throwable $e) {
        }

        try {
            if ($securityContext) {
                $created = $parent->addSubFolder($fields, $securityContext);
                if ($created instanceof Folder) {
                    return $created;
                }
                if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof Folder) {
                    return $created['OBJECT'];
                }
            }
        } catch (Throwable $e) {
        }

        throw new RuntimeException('Cannot create Disk folder: ' . $name);
    }
}

if (!function_exists('sb_disk_get_or_create_sitebuilder_root')) {
    function sb_disk_get_or_create_sitebuilder_root(): Folder
    {
        $root = sb_disk_root_folder();

        $folder = sb_disk_find_child_folder_by_name($root, 'SiteBuilder');
        if ($folder) {
            return $folder;
        }

        return sb_disk_add_subfolder($root, 'SiteBuilder');
    }
}

if (!function_exists('sb_disk_load_folder_by_id')) {
    function sb_disk_load_folder_by_id(int $folderId): ?Folder
    {
        if ($folderId <= 0) {
            return null;
        }

        sb_disk_require_module();

        $folder = Folder::loadById($folderId);
        return $folder instanceof Folder ? $folder : null;
    }
}

if (!function_exists('sb_disk_load_file_by_id')) {
    function sb_disk_load_file_by_id(int $fileId): ?File
    {
        if ($fileId <= 0) {
            return null;
        }

        sb_disk_require_module();

        $file = File::loadById($fileId);
        return $file instanceof File ? $file : null;
    }
}

if (!function_exists('sb_disk_site_folder_name')) {
    function sb_disk_site_folder_name(array $site): string
    {
        $slug = trim((string)($site['slug'] ?? ''));
        if ($slug !== '') {
            return $slug;
        }

        $id = (int)($site['id'] ?? 0);
        return $id > 0 ? ('site-' . $id) : 'sitebuilder-site';
    }
}

if (!function_exists('sb_disk_ensure_site_folder')) {
    function sb_disk_ensure_site_folder(int $siteId): Folder
    {
        $site = sb_find_site($siteId);
        if (!$site) {
            throw new RuntimeException('Site not found');
        }

        $folderId = (int)($site['diskFolderId'] ?? 0);
        if ($folderId > 0) {
            $folder = sb_disk_load_folder_by_id($folderId);
            if ($folder) {
                return $folder;
            }
        }

        $sitebuilderRoot = sb_disk_get_or_create_sitebuilder_root();
        $folderName = sb_disk_site_folder_name($site);

        $folder = sb_disk_find_child_folder_by_name($sitebuilderRoot, $folderName);
        if (!$folder) {
            $folder = sb_disk_add_subfolder($sitebuilderRoot, $folderName);
        }

        $sites = sb_read_sites();
        foreach ($sites as &$s) {
            if ((int)($s['id'] ?? 0) === $siteId) {
                $s['diskFolderId'] = (int)$folder->getId();
                $s['updatedAt'] = date('c');
                break;
            }
        }
        unset($s);
        sb_write_sites($sites);

        return $folder;
    }
}

if (!function_exists('sb_disk_upload_file_to_folder')) {
    function sb_disk_upload_file_to_folder(Folder $folder, array $fileArray): File
    {
        $securityContext = sb_disk_security_context();

        $uploadData = [
            'NAME' => (string)($fileArray['name'] ?? 'file'),
            'FILE_SIZE' => (int)($fileArray['size'] ?? 0),
            'TMP_NAME' => (string)($fileArray['tmp_name'] ?? ''),
            'TYPE' => (string)($fileArray['type'] ?? 'application/octet-stream'),
        ];

        try {
            $created = $folder->uploadFile($fileArray, []);
            if ($created instanceof File) {
                return $created;
            }
            if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                return $created['OBJECT'];
            }
        } catch (Throwable $e) {
        }

        try {
            if ($securityContext) {
                $created = $folder->uploadFile($fileArray, $securityContext);
                if ($created instanceof File) {
                    return $created;
                }
                if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                    return $created['OBJECT'];
                }
            }
        } catch (Throwable $e) {
        }

        try {
            $created = $folder->uploadFile($uploadData, []);
            if ($created instanceof File) {
                return $created;
            }
            if (is_array($created) && isset($created['OBJECT']) && $created['OBJECT'] instanceof File) {
                return $created['OBJECT'];
            }
        } catch (Throwable $e) {
        }

        throw new RuntimeException('Cannot upload file to Disk folder');
    }
}

if (!function_exists('sb_disk_file_belongs_to_site')) {
    function sb_disk_file_belongs_to_site(int $siteId, int $fileId): bool
    {
        $site = sb_find_site($siteId);
        if (!$site) {
            return false;
        }

        $siteFolderId = (int)($site['diskFolderId'] ?? 0);
        if ($siteFolderId <= 0) {
            return false;
        }

        $file = sb_disk_load_file_by_id($fileId);
        if (!$file) {
            return false;
        }

        $parentId = (int)$file->getParentId();
        return $parentId === $siteFolderId;
    }
}

if (!function_exists('sb_disk_delete_file')) {
    function sb_disk_delete_file(File $file): bool
    {
        $securityContext = sb_disk_security_context();

        try {
            if ($securityContext) {
                return (bool)$file->delete($securityContext);
            }
        } catch (Throwable $e) {
        }

        try {
            return (bool)$file->delete();
        } catch (Throwable $e) {
            return false;
        }
    }
}

if (!function_exists('sb_disk_file_download_url')) {
    function sb_disk_file_download_url(File $file): string
    {
        $fileId = (int)$file->getId();
        return '/bitrix/tools/disk/downloadFile.php?objectId=' . $fileId;
    }
}


---

2. Обнови /local/sitebuilder/api/bootstrap.php

Нужно подключить disk.php.

Полная версия файла:

<?php

define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;
use Bitrix\Disk\Storage;
use Bitrix\Disk\Folder;
use Bitrix\Disk\File;
use Bitrix\Disk\Driver;

global $USER;

header('Content-Type: application/json; charset=UTF-8');

if (!$USER->IsAuthorized()) {
    http_response_code(401);
    echo json_encode(['ok' => false, 'error' => 'NOT_AUTHORIZED'], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['ok' => false, 'error' => 'METHOD_NOT_ALLOWED'], JSON_UNESCAPED_UNICODE);
    exit;
}

if (!check_bitrix_sessid()) {
    http_response_code(403);
    echo json_encode(['ok' => false, 'error' => 'BAD_SESSID'], JSON_UNESCAPED_UNICODE);
    exit;
}

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/json.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/response.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/access.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/helpers.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/disk.php';


---

3. Полностью замени /local/sitebuilder/api/handlers/file.php

<?php

global $USER;

if ($action === 'file.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_viewer($siteId);

    try {
        $folder = sb_disk_ensure_site_folder($siteId);
        $children = sb_disk_get_children($folder);

        $files = [];
        foreach ($children as $child) {
            if (!$child instanceof \Bitrix\Disk\File) {
                continue;
            }

            $files[] = [
                'id' => (int)$child->getId(),
                'name' => (string)$child->getName(),
                'size' => (int)$child->getSize(),
                'createTime' => method_exists($child, 'getCreateTime') && $child->getCreateTime()
                    ? $child->getCreateTime()->format('c')
                    : '',
                'updateTime' => method_exists($child, 'getUpdateTime') && $child->getUpdateTime()
                    ? $child->getUpdateTime()->format('c')
                    : '',
                'downloadUrl' => sb_disk_file_download_url($child),
            ];
        }

        usort($files, static function ($a, $b) {
            return strcmp((string)$a['name'], (string)$b['name']);
        });

        sb_json_ok([
            'files' => $files,
            'folderId' => (int)$folder->getId(),
        ]);
    } catch (Throwable $e) {
        sb_json_error('DISK_ERROR', 500, [
            'message' => $e->getMessage(),
        ]);
    }
}

if ($action === 'file.upload') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_editor($siteId);

    if (empty($_FILES['file']) || !is_array($_FILES['file'])) {
        sb_json_error('FILE_REQUIRED', 422);
    }

    $upload = $_FILES['file'];

    if ((int)($upload['error'] ?? UPLOAD_ERR_NO_FILE) !== UPLOAD_ERR_OK) {
        sb_json_error('UPLOAD_ERROR', 422, [
            'phpUploadError' => (int)($upload['error'] ?? UPLOAD_ERR_NO_FILE),
        ]);
    }

    if (!is_uploaded_file((string)($upload['tmp_name'] ?? ''))) {
        sb_json_error('BAD_UPLOADED_FILE', 422);
    }

    try {
        $folder = sb_disk_ensure_site_folder($siteId);
        $file = sb_disk_upload_file_to_folder($folder, $upload);

        sb_json_ok([
            'file' => [
                'id' => (int)$file->getId(),
                'name' => (string)$file->getName(),
                'size' => (int)$file->getSize(),
                'downloadUrl' => sb_disk_file_download_url($file),
            ],
            'folderId' => (int)$folder->getId(),
        ]);
    } catch (Throwable $e) {
        sb_json_error('DISK_ERROR', 500, [
            'message' => $e->getMessage(),
        ]);
    }
}

if ($action === 'file.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $fileId = (int)($_POST['fileId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }
    if ($fileId <= 0) {
        sb_json_error('FILE_ID_REQUIRED', 422);
    }

    sb_require_editor($siteId);

    try {
        if (!sb_disk_file_belongs_to_site($siteId, $fileId)) {
            sb_json_error('FILE_NOT_IN_SITE', 422);
        }

        $file = sb_disk_load_file_by_id($fileId);
        if (!$file) {
            sb_json_error('FILE_NOT_FOUND', 404);
        }

        $ok = sb_disk_delete_file($file);
        if (!$ok) {
            sb_json_error('DELETE_FAILED', 500);
        }

        sb_json_ok();
    } catch (Throwable $e) {
        sb_json_error('DISK_ERROR', 500, [
            'message' => $e->getMessage(),
        ]);
    }
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'file',
    'action' => $action,
]);


---

4. Что теперь должно работать

После вставки должны работать:

file.list

file.upload

file.delete



---

5. Что происходит при первом file.list

Если у сайта ещё нет diskFolderId, то при первом вызове:

ищется общее Disk-хранилище

в корне ищется папка SiteBuilder

если её нет — создаётся

внутри ищется папка сайта по slug

если её нет — создаётся

diskFolderId записывается в sites.json


То есть инициализация идёт автоматически.


---

6. Что проверить руками

Проверяй в таком порядке:

1. file.list

Для сайта должен:

либо создать папку

либо вернуть пустой список

либо вернуть существующие файлы


2. file.upload

Загрузи любой тестовый файл.

3. снова file.list

Должен вернуть уже загруженный файл.

4. file.delete

Удалить этот файл.


---

7. Важные нюансы

Удаление сейчас разрешено только для файлов текущего сайта

Проверка идёт по parentId === site.diskFolderId.

Это простой и безопасный вариант.

Подпапки сайта пока не поддержаны

Сейчас всё хранится в одной папке сайта. Это нормально для первого этапа.

Права Disk пока не синхронизируем

То есть сейчас мы подняли рабочую базу file API, но не делали ещё синхронизацию ролей OWNER / ADMIN / EDITOR / VIEWER в права Disk-объектов.

Это лучше добавить отдельным шагом, потому что в разных коробках Bitrix API прав может отличаться.


---

8. Если на твоей коробке Disk начнёт ругаться

На коробках Bitrix бывают разные сигнатуры методов. Я уже заложил несколько fallback-вариантов, но если увидишь что-то вроде:

Call to undefined method Bitrix\Disk\Storage::loadByEntity()

Argument #2 must be of type array

Too few arguments to function getChildren()

Cannot create Disk folder

Cannot upload file to Disk folder


тогда просто пришли мне точный текст ошибки, и я подправлю lib/disk.php именно под твою версию коробки.


---

9. Что у тебя уже закрыто по API

На этом этапе у тебя уже есть рабочие ветки:

site.*

page.*

block.*

menu.*

access.*

template.*

layout.*

file.*


То есть backend-каркас уже собран.


---

10. Что делать дальше правильно

Теперь следующий логичный этап уже не в API, а в входных страницах и фронте:

1. сделать index.php


2. сделать страницу управления страницами/сайтами


3. сделать editor.php


4. вынести JS редактора по файлам


5. потом layout.php


6. потом public.php



Я бы следующим сообщением начал уже с рабочей страницы index.php, которая:

показывает список сайтов

создаёт сайт

удаляет сайт

открывает editor/layout/menu/files/settings


Напиши: давай index.php