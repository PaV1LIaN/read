Ок, кодом. Давай сделаем без замены всего огромного script.js: ниже даю точные готовые вставки, чтобы не сломать текущий рабочий диск.


---

1. components/disk/api.php

Файл:

/local/sitebuilder/components/disk/api.php

Замени целиком:

<?php

require_once __DIR__ . '/bootstrap.php';

try {
    $action = $_GET['action'] ?? '';

    switch ($action) {
        case 'resolveRoot':
            require __DIR__ . '/actions/resolve_root.php';
            break;

        case 'getSettings':
            require __DIR__ . '/actions/get_settings.php';
            break;

        case 'saveSettings':
            require __DIR__ . '/actions/save_settings.php';
            break;

        case 'getPermissions':
            require __DIR__ . '/actions/get_permissions.php';
            break;

        case 'getRootOptions':
            require __DIR__ . '/actions/get_root_options.php';
            break;

        case 'list':
            require __DIR__ . '/actions/list.php';
            break;

        case 'upload':
            require __DIR__ . '/actions/upload.php';
            break;

        case 'createFolder':
            require __DIR__ . '/actions/create_folder.php';
            break;

        case 'rename':
            require __DIR__ . '/actions/rename.php';
            break;

        case 'delete':
            require __DIR__ . '/actions/delete.php';
            break;

        case 'move':
            require __DIR__ . '/actions/move.php';
            break;

        case 'copy':
            require __DIR__ . '/actions/copy.php';
            break;

        case 'search':
            require __DIR__ . '/actions/search.php';
            break;

        case 'download':
            require __DIR__ . '/actions/download.php';
            break;

        case 'unpackArchive':
            require __DIR__ . '/actions/unpack_archive.php';
            break;

        case 'initSiteRoot':
            require __DIR__ . '/actions/init_site_root.php';
            break;

        case 'initBlockRoot':
            require __DIR__ . '/actions/init_block_root.php';
            break;

        case 'bootstrap':
            require __DIR__ . '/actions/bootstrap.php';
            break;

        default:
            DiskResponse::error('UNKNOWN_ACTION', 'Неизвестное действие');
    }
} catch (Throwable $e) {
    DiskResponse::error('SERVER_ERROR', $e->getMessage());
}


---

2. components/disk/actions/unpack_archive.php

Создай файл:

/local/sitebuilder/components/disk/actions/unpack_archive.php

Вставь целиком:

<?php

use Bitrix\Disk\Driver;
use Bitrix\Disk\File;
use Bitrix\Disk\Folder;

DiskCsrf::validateFromRequest();

$data = disk_read_json_body();
$currentUserId = DiskCurrentUser::requireId();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => $currentUserId,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $settings);
$permissions = DiskPermissionService::resolve($context, $settings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canView');
DiskValidator::assertCan($permissions, 'canUpload');

if ($rootFolderId === null || $rootFolderId <= 0) {
    throw new RuntimeException('ROOT_FOLDER_NOT_RESOLVED');
}

$fileId = (int)($data['fileId'] ?? 0);
$currentFolderId = (int)($data['currentFolderId'] ?? 0);

if ($fileId <= 0) {
    throw new RuntimeException('INVALID_FILE_ID');
}

if ($currentFolderId <= 0) {
    throw new RuntimeException('INVALID_FOLDER_ID');
}

DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId, $context);

$file = File::loadById($fileId);

if (!$file instanceof File) {
    throw new RuntimeException('DISK_FILE_NOT_FOUND');
}

$sourceParentId = (int)$file->getParentId();
DiskValidator::assertFolderInsideRoot($sourceParentId, $rootFolderId, $context);

$extension = mb_strtolower((string)$file->getExtension());

if ($extension !== 'zip') {
    throw new RuntimeException('ONLY_ZIP_SUPPORTED');
}

if (!class_exists('ZipArchive')) {
    throw new RuntimeException('ZIP_EXTENSION_NOT_INSTALLED');
}

$zipPath = sb_disk_archive_get_file_path($file);

$zip = new ZipArchive();
$openResult = $zip->open($zipPath);

if ($openResult !== true) {
    throw new RuntimeException('ZIP_OPEN_ERROR');
}

$securityContext = Driver::getInstance()->getFakeSecurityContext($context->currentUserId);
$currentFolder = Folder::loadById($currentFolderId);

if (!$currentFolder instanceof Folder) {
    $zip->close();
    throw new RuntimeException('DISK_FOLDER_NOT_FOUND');
}

$archiveBaseName = pathinfo((string)$file->getName(), PATHINFO_FILENAME);
$archiveBaseName = DiskNameSanitizer::sanitizeFolderName($archiveBaseName, 'Распакованный архив');

$targetFolderName = sb_disk_archive_unique_name($currentFolder, $archiveBaseName, $securityContext);

$targetFolder = $currentFolder->addSubFolder([
    'NAME' => $targetFolderName,
    'CREATED_BY' => $context->currentUserId,
], [], true);

if (!$targetFolder instanceof Folder) {
    $zip->close();
    throw new RuntimeException('CREATE_TARGET_FOLDER_ERROR');
}

$allowedExtensions = $settings['allowedExtensions'] ?? [];
$allowedExtensions = is_array($allowedExtensions) ? array_map('mb_strtolower', $allowedExtensions) : [];

$maxFileSize = (int)($settings['maxFileSize'] ?? 0);

$maxFiles = 500;
$maxTotalSize = 512 * 1024 * 1024;

$totalSize = 0;
$extractedFiles = 0;
$createdFolders = 1;

$folderCache = [
    '' => $targetFolder,
];

try {
    for ($i = 0; $i < $zip->numFiles; $i++) {
        $stat = $zip->statIndex($i);

        if (!is_array($stat)) {
            continue;
        }

        $rawName = (string)($stat['name'] ?? '');

        if ($rawName === '') {
            continue;
        }

        $rawName = str_replace('\\', '/', $rawName);
        $rawName = trim($rawName);

        if ($rawName === '' || str_starts_with($rawName, '__MACOSX/')) {
            continue;
        }

        if (basename($rawName) === '.DS_Store') {
            continue;
        }

        $isDir = str_ends_with($rawName, '/');
        $pathParts = sb_disk_archive_safe_path_parts($rawName);

        if (empty($pathParts)) {
            continue;
        }

        if ($isDir) {
            [, $newFolders] = sb_disk_archive_ensure_folder_path(
                $context,
                $securityContext,
                $targetFolder,
                $folderCache,
                $pathParts
            );

            $createdFolders += $newFolders;
            continue;
        }

        if ($extractedFiles >= $maxFiles) {
            throw new RuntimeException('ARCHIVE_TOO_MANY_FILES');
        }

        $fileSize = (int)($stat['size'] ?? 0);

        if ($fileSize < 0) {
            $fileSize = 0;
        }

        if ($maxFileSize > 0 && $fileSize > $maxFileSize) {
            throw new RuntimeException('ARCHIVE_FILE_TOO_LARGE: ' . $rawName);
        }

        $totalSize += $fileSize;

        if ($totalSize > $maxTotalSize) {
            throw new RuntimeException('ARCHIVE_TOTAL_SIZE_TOO_LARGE');
        }

        $fileName = array_pop($pathParts);
        $safeFileName = DiskNameSanitizer::sanitizeFolderName($fileName, 'file');

        if (!empty($allowedExtensions)) {
            $fileExt = mb_strtolower(pathinfo($safeFileName, PATHINFO_EXTENSION));

            if (!in_array($fileExt, $allowedExtensions, true)) {
                throw new RuntimeException('EXTENSION_NOT_ALLOWED_IN_ARCHIVE: ' . $safeFileName);
            }
        }

        [$destinationFolder, $newFolders] = sb_disk_archive_ensure_folder_path(
            $context,
            $securityContext,
            $targetFolder,
            $folderCache,
            $pathParts
        );

        $createdFolders += $newFolders;

        $safeFileName = sb_disk_archive_unique_name($destinationFolder, $safeFileName, $securityContext);

        $stream = $zip->getStream($rawName);

        if (!is_resource($stream)) {
            throw new RuntimeException('ZIP_READ_ENTRY_ERROR: ' . $rawName);
        }

        $tmpFile = tempnam(sys_get_temp_dir(), 'sb_zip_');

        if ($tmpFile === false) {
            fclose($stream);
            throw new RuntimeException('TEMP_FILE_CREATE_ERROR');
        }

        $out = fopen($tmpFile, 'wb');

        if (!is_resource($out)) {
            fclose($stream);
            @unlink($tmpFile);
            throw new RuntimeException('TEMP_FILE_OPEN_ERROR');
        }

        stream_copy_to_stream($stream, $out);

        fclose($stream);
        fclose($out);

        $uploadFile = [
            'name' => $safeFileName,
            'type' => sb_disk_archive_detect_mime($safeFileName),
            'tmp_name' => $tmpFile,
            'error' => UPLOAD_ERR_OK,
            'size' => filesize($tmpFile) ?: 0,
        ];

        $createdFile = $destinationFolder->uploadFile(
            $uploadFile,
            [
                'NAME' => $safeFileName,
                'CREATED_BY' => $context->currentUserId,
            ],
            []
        );

        @unlink($tmpFile);

        if (!$createdFile instanceof File) {
            throw new RuntimeException('DISK_UPLOAD_EXTRACTED_FILE_ERROR: ' . $safeFileName);
        }

        $extractedFiles++;
    }
} finally {
    $zip->close();
}

DiskResponse::success([
    'targetFolder' => [
        'id' => (int)$targetFolder->getId(),
        'name' => (string)$targetFolder->getName(),
    ],
    'extractedFiles' => $extractedFiles,
    'createdFolders' => $createdFolders,
    'totalSize' => $totalSize,
]);

function sb_disk_archive_get_file_path(File $file): string
{
    $fileId = 0;

    if (method_exists($file, 'getFileId')) {
        $fileId = (int)$file->getFileId();
    }

    if ($fileId <= 0 && method_exists($file, 'getFile')) {
        $fileData = $file->getFile();

        if (is_array($fileData)) {
            $fileId = (int)($fileData['ID'] ?? 0);
        }
    }

    if ($fileId <= 0) {
        throw new RuntimeException('BITRIX_FILE_ID_NOT_FOUND');
    }

    $fileArray = CFile::MakeFileArray($fileId);

    if (!is_array($fileArray)) {
        throw new RuntimeException('BITRIX_FILE_ARRAY_NOT_FOUND');
    }

    $path = (string)($fileArray['tmp_name'] ?? '');

    if ($path === '' || !is_file($path)) {
        throw new RuntimeException('BITRIX_FILE_PATH_NOT_FOUND');
    }

    return $path;
}

function sb_disk_archive_safe_path_parts(string $path): array
{
    $path = str_replace('\\', '/', $path);
    $path = trim($path, "/ \t\n\r\0\x0B");

    if ($path === '') {
        return [];
    }

    if (preg_match('~(^|/)\.\.($|/)~', $path)) {
        throw new RuntimeException('ARCHIVE_UNSAFE_PATH');
    }

    if (preg_match('~^[a-zA-Z]:~', $path)) {
        throw new RuntimeException('ARCHIVE_UNSAFE_PATH');
    }

    $parts = explode('/', $path);
    $safeParts = [];

    foreach ($parts as $part) {
        $part = trim($part);

        if ($part === '' || $part === '.' || $part === '..') {
            continue;
        }

        $safeParts[] = DiskNameSanitizer::sanitizeFolderName($part, 'item');
    }

    return $safeParts;
}

function sb_disk_archive_ensure_folder_path(
    DiskContext $context,
    $securityContext,
    Folder $rootFolder,
    array &$folderCache,
    array $pathParts
): array {
    if (empty($pathParts)) {
        return [$rootFolder, 0];
    }

    $current = $rootFolder;
    $cacheKey = '';
    $createdCount = 0;

    foreach ($pathParts as $part) {
        $part = DiskNameSanitizer::sanitizeFolderName($part, 'Папка');
        $cacheKey = $cacheKey === '' ? $part : $cacheKey . '/' . $part;

        if (isset($folderCache[$cacheKey]) && $folderCache[$cacheKey] instanceof Folder) {
            $current = $folderCache[$cacheKey];
            continue;
        }

        $existing = sb_disk_archive_find_child_folder($current, $part, $securityContext);

        if ($existing instanceof Folder) {
            $current = $existing;
            $folderCache[$cacheKey] = $current;
            continue;
        }

        $created = $current->addSubFolder([
            'NAME' => $part,
            'CREATED_BY' => $context->currentUserId,
        ], [], true);

        if (!$created instanceof Folder) {
            throw new RuntimeException('DISK_CREATE_EXTRACT_FOLDER_ERROR: ' . $part);
        }

        $createdCount++;
        $current = $created;
        $folderCache[$cacheKey] = $current;
    }

    return [$current, $createdCount];
}

function sb_disk_archive_find_child_folder(Folder $parent, string $name, $securityContext): ?Folder
{
    $children = $parent->getChildren($securityContext);

    foreach ($children as $child) {
        if ($child instanceof Folder && (string)$child->getName() === $name) {
            return $child;
        }
    }

    return null;
}

function sb_disk_archive_name_exists(Folder $parent, string $name, $securityContext): bool
{
    $children = $parent->getChildren($securityContext);

    foreach ($children as $child) {
        if ((string)$child->getName() === $name) {
            return true;
        }
    }

    return false;
}

function sb_disk_archive_unique_name(Folder $parent, string $name, $securityContext): string
{
    $name = DiskNameSanitizer::sanitizeFolderName($name, 'item');

    if (!sb_disk_archive_name_exists($parent, $name, $securityContext)) {
        return $name;
    }

    $extension = pathinfo($name, PATHINFO_EXTENSION);
    $baseName = $extension !== ''
        ? mb_substr($name, 0, -(mb_strlen($extension) + 1))
        : $name;

    for ($i = 1; $i <= 999; $i++) {
        $candidate = $extension !== ''
            ? $baseName . ' (' . $i . ').' . $extension
            : $baseName . ' (' . $i . ')';

        if (!sb_disk_archive_name_exists($parent, $candidate, $securityContext)) {
            return $candidate;
        }
    }

    return $baseName . ' (' . time() . ')' . ($extension !== '' ? '.' . $extension : '');
}

function sb_disk_archive_detect_mime(string $name): string
{
    $ext = mb_strtolower(pathinfo($name, PATHINFO_EXTENSION));

    $map = [
        'txt' => 'text/plain',
        'csv' => 'text/csv',
        'json' => 'application/json',
        'xml' => 'application/xml',
        'pdf' => 'application/pdf',
        'doc' => 'application/msword',
        'docx' => 'application/vnd.openxmlformats-officedocument.wordprocessingml.document',
        'xls' => 'application/vnd.ms-excel',
        'xlsx' => 'application/vnd.openxmlformats-officedocument.spreadsheetml.sheet',
        'ppt' => 'application/vnd.ms-powerpoint',
        'pptx' => 'application/vnd.openxmlformats-officedocument.presentationml.presentation',
        'jpg' => 'image/jpeg',
        'jpeg' => 'image/jpeg',
        'png' => 'image/png',
        'gif' => 'image/gif',
        'webp' => 'image/webp',
        'svg' => 'image/svg+xml',
        'zip' => 'application/zip',
    ];

    return $map[$ext] ?? 'application/octet-stream';
}


---

3. Правки в components/disk/script.js

Файл:

/local/sitebuilder/components/disk/script.js

Тут лучше не менять весь файл. Вставь 3 готовых куска.


---

3.1. Добавь функцию проверки ZIP

В самый низ файла, перед функцией formatBytes, вставь:

function isArchiveItem(item) {
  if (!item || item.entityType !== 'file') {
    return false;
  }

  var extension = String(item.extension || '').toLowerCase();
  var name = String(item.name || '').toLowerCase();

  return extension === 'zip' || name.slice(-4) === '.zip';
}

Должно получиться примерно так:

function isArchiveItem(item) {
  if (!item || item.entityType !== 'file') {
    return false;
  }

  var extension = String(item.extension || '').toLowerCase();
  var name = String(item.name || '').toLowerCase();

  return extension === 'zip' || name.slice(-4) === '.zip';
}

function formatBytes(bytes) {
    ...
}


---

3.2. Добавь кнопку в список/таблицу

Найди в renderItemsTable() кусок:

(item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +

Замени на:

(item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
(isArchiveItem(item)
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="unpack">Распаковать</button>'
  : '') +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +


---

3.3. Добавь кнопку в плитки

Найди в renderItemsGrid() такой же кусок:

(item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +

Замени на:

(item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
(isArchiveItem(item)
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="unpack">Распаковать</button>'
  : '') +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +


---

3.4. Добавь обработчик кнопки

Найди в script.js блок скачивания:

var downloadBtn = e.target.closest('[data-row-action="download"]');
if (downloadBtn) {

После всего блока download, до блока rename, вставь:

var unpackBtn = e.target.closest('[data-row-action="unpack"]');

if (unpackBtn) {
  var unpackRow = e.target.closest('[data-id][data-entity-type="file"]');

  if (!unpackRow) {
    return;
  }

  var fileName = unpackRow.getAttribute('data-name') || 'архив';

  var confirmUnpack = window.confirm(
    'Распаковать архив "' + fileName + '"?\n\n' +
    'Будет создана новая папка с содержимым архива.'
  );

  if (!confirmUnpack) {
    return;
  }

  try {
    self.setLoading(true);

    var unpackPayload = self.getBasePayload();

    unpackPayload.fileId = Number(unpackRow.getAttribute('data-id') || 0);
    unpackPayload.currentFolderId = self.state.currentFolderId || self.state.rootFolderId;
    unpackPayload.sessid = self.getSessid();

    var unpackRes = await self.api('unpackArchive', unpackPayload);

    if (!unpackRes || !unpackRes.ok) {
      window.alert((unpackRes && (unpackRes.message || unpackRes.error)) || 'Ошибка распаковки');
      return;
    }

    var targetFolder = unpackRes.data && unpackRes.data.targetFolder
      ? unpackRes.data.targetFolder
      : null;

    if (targetFolder && targetFolder.id) {
      await self.loadFolder(Number(targetFolder.id));
    } else {
      await self.loadFolder(self.state.currentFolderId || self.state.rootFolderId);
    }
  } catch (err) {
    console.error(err);
    window.alert(err.message || 'Ошибка распаковки');
  } finally {
    self.setLoading(false);
  }

  return;
}


---

4. Важно: строка файла должна иметь data-name

В script.js в рендере строки должно быть что-то похожее:

'<tr data-id="' + escapeHtml(item.id) + '" data-entity-type="' + escapeHtml(item.entityType) + '">'

Нужно заменить на:

'<tr data-id="' + escapeHtml(item.id) + '" data-entity-type="' + escapeHtml(item.entityType) + '" data-name="' + escapeHtml(item.name || '') + '">'

И в плитке тоже должно быть что-то похожее:

'<article class="sb-disk__card" data-id="' + escapeHtml(item.id) + '" data-entity-type="' + escapeHtml(item.entityType) + '">'

Заменить на:

'<article class="sb-disk__card" data-id="' + escapeHtml(item.id) + '" data-entity-type="' + escapeHtml(item.entityType) + '" data-name="' + escapeHtml(item.name || '') + '">'


---

После этого:

1. Загрузи .zip.
2. У него появится кнопка “Распаковать”.
3. Нажми.
4. Создастся папка с названием архива.

Если выйдет ошибка:

ZIP_EXTENSION_NOT_INSTALLED

значит надо поставить модуль PHP ZIP:

sudo apt install php-zip
sudo systemctl restart php8.1-fpm

Если у тебя PHP не 8.1, рестарт нужен для твоей версии PHP-FPM.