Да, нормальное решение здесь — не пытаться распаковать весь архив одним запросом. У тебя сейчас 504 Gateway Timeout, потому что сервер не успевает распаковать архив до таймаута Angie/Nginx.

Сделаем правильно:

1. Нажал “Распаковать”.
2. JS создаёт задачу распаковки.
3. Потом много маленьких запросов распаковывают архив частями.
4. Модальное окно показывает реальный прогресс.
5. 504 больше не должно быть.


---

1. В api.php добавь 2 action

Файл:

/local/sitebuilder/components/disk/api.php

В switch ($action) добавь рядом с unpackArchive:

case 'unpackArchiveStart':
    require __DIR__ . '/actions/unpack_archive_start.php';
    break;

case 'unpackArchiveStep':
    require __DIR__ . '/actions/unpack_archive_step.php';
    break;

Должно быть примерно так:

case 'unpackArchive':
    require __DIR__ . '/actions/unpack_archive.php';
    break;

case 'unpackArchiveStart':
    require __DIR__ . '/actions/unpack_archive_start.php';
    break;

case 'unpackArchiveStep':
    require __DIR__ . '/actions/unpack_archive_step.php';
    break;

Старый unpackArchive пока не удаляй.


---

2. Создай общий файл unpack_archive_lib.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_lib.php

Код:

<?php

use Bitrix\Disk\File;
use Bitrix\Disk\Folder;

if (!function_exists('sb_disk_unpack_jobs_dir')) {
    function sb_disk_unpack_jobs_dir(): string
    {
        $dir = rtrim((string)$_SERVER['DOCUMENT_ROOT'], '/') . '/upload/sitebuilder/disk_unpack_jobs';

        if (!is_dir($dir)) {
            mkdir($dir, 0775, true);
        }

        return $dir;
    }
}

if (!function_exists('sb_disk_unpack_job_path')) {
    function sb_disk_unpack_job_path(string $jobId): string
    {
        $jobId = preg_replace('/[^a-f0-9]/i', '', $jobId);

        if ($jobId === '') {
            throw new RuntimeException('INVALID_UNPACK_JOB_ID');
        }

        return sb_disk_unpack_jobs_dir() . '/' . $jobId . '.json';
    }
}

if (!function_exists('sb_disk_unpack_save_job')) {
    function sb_disk_unpack_save_job(array $job): void
    {
        $jobId = (string)($job['id'] ?? '');

        if ($jobId === '') {
            throw new RuntimeException('EMPTY_UNPACK_JOB_ID');
        }

        file_put_contents(
            sb_disk_unpack_job_path($jobId),
            json_encode($job, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES)
        );
    }
}

if (!function_exists('sb_disk_unpack_load_job')) {
    function sb_disk_unpack_load_job(string $jobId): array
    {
        $path = sb_disk_unpack_job_path($jobId);

        if (!is_file($path)) {
            throw new RuntimeException('UNPACK_JOB_NOT_FOUND');
        }

        $json = json_decode((string)file_get_contents($path), true);

        if (!is_array($json)) {
            throw new RuntimeException('UNPACK_JOB_BROKEN');
        }

        return $json;
    }
}

if (!function_exists('sb_disk_unpack_delete_job')) {
    function sb_disk_unpack_delete_job(string $jobId): void
    {
        $path = sb_disk_unpack_job_path($jobId);

        if (is_file($path)) {
            @unlink($path);
        }
    }
}

if (!function_exists('sb_disk_unpack_sanitize_name')) {
    function sb_disk_unpack_sanitize_name(string $name, string $fallback = 'item'): string
    {
        $name = trim($name);

        if ($name === '') {
            $name = $fallback;
        }

        if (class_exists('DiskNameSanitizer') && method_exists('DiskNameSanitizer', 'sanitizeFolderName')) {
            return DiskNameSanitizer::sanitizeFolderName($name, $fallback);
        }

        $name = str_replace(["\0", '/', '\\'], '_', $name);
        $name = trim($name);

        return $name !== '' ? $name : $fallback;
    }
}

if (!function_exists('sb_disk_unpack_get_file_path')) {
    function sb_disk_unpack_get_file_path(File $file): string
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
}

if (!function_exists('sb_disk_unpack_safe_path_parts')) {
    function sb_disk_unpack_safe_path_parts(string $path): array
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

            $safeParts[] = sb_disk_unpack_sanitize_name($part, 'item');
        }

        return $safeParts;
    }
}

if (!function_exists('sb_disk_unpack_find_child_folder')) {
    function sb_disk_unpack_find_child_folder(Folder $parent, string $name, $securityContext): ?Folder
    {
        $children = $parent->getChildren($securityContext);

        foreach ($children as $child) {
            if ($child instanceof Folder && (string)$child->getName() === $name) {
                return $child;
            }
        }

        return null;
    }
}

if (!function_exists('sb_disk_unpack_name_exists')) {
    function sb_disk_unpack_name_exists(Folder $parent, string $name, $securityContext): bool
    {
        $children = $parent->getChildren($securityContext);

        foreach ($children as $child) {
            if ((string)$child->getName() === $name) {
                return true;
            }
        }

        return false;
    }
}

if (!function_exists('sb_disk_unpack_unique_name')) {
    function sb_disk_unpack_unique_name(Folder $parent, string $name, $securityContext): string
    {
        $name = sb_disk_unpack_sanitize_name($name, 'item');

        if (!sb_disk_unpack_name_exists($parent, $name, $securityContext)) {
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

            if (!sb_disk_unpack_name_exists($parent, $candidate, $securityContext)) {
                return $candidate;
            }
        }

        return $baseName . ' (' . time() . ')' . ($extension !== '' ? '.' . $extension : '');
    }
}

if (!function_exists('sb_disk_unpack_ensure_folder_path')) {
    function sb_disk_unpack_ensure_folder_path(
        DiskContext $context,
        $securityContext,
        Folder $rootFolder,
        array $pathParts
    ): array {
        if (empty($pathParts)) {
            return [$rootFolder, 0];
        }

        $current = $rootFolder;
        $createdCount = 0;

        foreach ($pathParts as $part) {
            $part = sb_disk_unpack_sanitize_name($part, 'Папка');

            $existing = sb_disk_unpack_find_child_folder($current, $part, $securityContext);

            if ($existing instanceof Folder) {
                $current = $existing;
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
        }

        return [$current, $createdCount];
    }
}

if (!function_exists('sb_disk_unpack_detect_mime')) {
    function sb_disk_unpack_detect_mime(string $name): string
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
}


---

3. Создай unpack_archive_start.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_start.php

Код:

<?php

use Bitrix\Disk\File;
use Bitrix\Disk\Folder;

require_once __DIR__ . '/unpack_archive_lib.php';

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

if ($fileId <= 0) {
    throw new RuntimeException('INVALID_FILE_ID');
}

$file = File::loadById($fileId);

if (!$file instanceof File) {
    throw new RuntimeException('DISK_FILE_NOT_FOUND');
}

$sourceParentId = (int)$file->getParentId();

if ($sourceParentId <= 0) {
    throw new RuntimeException('ARCHIVE_PARENT_FOLDER_NOT_FOUND');
}

DiskValidator::assertFolderInsideRoot($sourceParentId, $rootFolderId, $context);

$targetFolder = Folder::loadById($sourceParentId);

if (!$targetFolder instanceof Folder) {
    throw new RuntimeException('ARCHIVE_PARENT_FOLDER_NOT_FOUND');
}

$extension = mb_strtolower((string)$file->getExtension());

if ($extension !== 'zip') {
    throw new RuntimeException('ONLY_ZIP_SUPPORTED');
}

if (!class_exists('ZipArchive')) {
    throw new RuntimeException('ZIP_EXTENSION_NOT_INSTALLED');
}

$zipPath = sb_disk_unpack_get_file_path($file);

$zip = new ZipArchive();
$openResult = $zip->open($zipPath);

if ($openResult !== true) {
    throw new RuntimeException('ZIP_OPEN_ERROR');
}

$allowedExtensions = $settings['allowedExtensions'] ?? [];
$allowedExtensions = is_array($allowedExtensions) ? array_map('mb_strtolower', $allowedExtensions) : [];

$maxFileSize = (int)($settings['maxFileSize'] ?? 0);

$maxFiles = 5000;
$maxTotalSize = 1024 * 1024 * 1024;

$totalSize = 0;
$totalFiles = 0;
$totalFolders = 0;
$entries = [];

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
        $pathParts = sb_disk_unpack_safe_path_parts($rawName);

        if (empty($pathParts)) {
            continue;
        }

        if ($isDir) {
            $totalFolders++;

            $entries[] = [
                'rawName' => $rawName,
                'isDir' => true,
                'size' => 0,
            ];

            continue;
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

        $fileName = end($pathParts);
        $safeFileName = sb_disk_unpack_sanitize_name((string)$fileName, 'file');

        if (!empty($allowedExtensions)) {
            $fileExt = mb_strtolower(pathinfo($safeFileName, PATHINFO_EXTENSION));

            if (!in_array($fileExt, $allowedExtensions, true)) {
                throw new RuntimeException('EXTENSION_NOT_ALLOWED_IN_ARCHIVE: ' . $safeFileName);
            }
        }

        $totalFiles++;

        if ($totalFiles > $maxFiles) {
            throw new RuntimeException('ARCHIVE_TOO_MANY_FILES');
        }

        $entries[] = [
            'rawName' => $rawName,
            'isDir' => false,
            'size' => $fileSize,
        ];
    }
} finally {
    $zip->close();
}

$jobId = bin2hex(random_bytes(16));

$job = [
    'id' => $jobId,
    'createdAt' => time(),

    'siteId' => $context->siteId,
    'pageId' => $context->pageId,
    'blockId' => $context->blockId,
    'currentUserId' => $context->currentUserId,

    'rootFolderId' => (int)$rootFolderId,
    'fileId' => (int)$file->getId(),
    'targetFolderId' => (int)$targetFolder->getId(),
    'archiveName' => (string)$file->getName(),

    'settings' => [
        'allowedExtensions' => $allowedExtensions,
        'maxFileSize' => $maxFileSize,
    ],

    'entries' => $entries,
    'index' => 0,

    'totalEntries' => count($entries),
    'totalFiles' => $totalFiles,
    'totalFolders' => $totalFolders,
    'totalSize' => $totalSize,

    'extractedFiles' => 0,
    'createdFolders' => 0,
    'processedSize' => 0,
];

sb_disk_unpack_save_job($job);

DiskResponse::success([
    'jobId' => $jobId,
    'archiveName' => (string)$file->getName(),
    'targetFolder' => [
        'id' => (int)$targetFolder->getId(),
        'name' => (string)$targetFolder->getName(),
    ],
    'totalEntries' => count($entries),
    'totalFiles' => $totalFiles,
    'totalFolders' => $totalFolders,
    'totalSize' => $totalSize,
]);


---

4. Создай unpack_archive_step.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_step.php

Код:

<?php

use Bitrix\Disk\Driver;
use Bitrix\Disk\File;
use Bitrix\Disk\Folder;

require_once __DIR__ . '/unpack_archive_lib.php';

DiskCsrf::validateFromRequest();

$data = disk_read_json_body();
$currentUserId = DiskCurrentUser::requireId();

$jobId = (string)($data['jobId'] ?? '');

if ($jobId === '') {
    throw new RuntimeException('INVALID_UNPACK_JOB_ID');
}

$job = sb_disk_unpack_load_job($jobId);

if ((int)($job['currentUserId'] ?? 0) !== $currentUserId) {
    throw new RuntimeException('UNPACK_JOB_ACCESS_DENIED');
}

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($job['siteId'] ?? 0),
    'pageId' => (int)($job['pageId'] ?? 0),
    'blockId' => (int)($job['blockId'] ?? 0),
    'currentUserId' => $currentUserId,
]);

DiskValidator::assertContext($context);

$rootFolderId = (int)($job['rootFolderId'] ?? 0);
$targetFolderId = (int)($job['targetFolderId'] ?? 0);
$fileId = (int)($job['fileId'] ?? 0);

if ($rootFolderId <= 0 || $targetFolderId <= 0 || $fileId <= 0) {
    throw new RuntimeException('UNPACK_JOB_INVALID_DATA');
}

DiskValidator::assertFolderInsideRoot($targetFolderId, $rootFolderId, $context);

$file = File::loadById($fileId);

if (!$file instanceof File) {
    throw new RuntimeException('DISK_FILE_NOT_FOUND');
}

$targetFolder = Folder::loadById($targetFolderId);

if (!$targetFolder instanceof Folder) {
    throw new RuntimeException('TARGET_FOLDER_NOT_FOUND');
}

if (!class_exists('ZipArchive')) {
    throw new RuntimeException('ZIP_EXTENSION_NOT_INSTALLED');
}

$zipPath = sb_disk_unpack_get_file_path($file);

$zip = new ZipArchive();
$openResult = $zip->open($zipPath);

if ($openResult !== true) {
    throw new RuntimeException('ZIP_OPEN_ERROR');
}

$securityContext = Driver::getInstance()->getFakeSecurityContext($context->currentUserId);

$entries = is_array($job['entries'] ?? null) ? $job['entries'] : [];
$totalEntries = count($entries);
$index = (int)($job['index'] ?? 0);

$settings = is_array($job['settings'] ?? null) ? $job['settings'] : [];
$allowedExtensions = is_array($settings['allowedExtensions'] ?? null)
    ? array_map('mb_strtolower', $settings['allowedExtensions'])
    : [];

$batchLimit = 5;
$startedAt = microtime(true);
$timeLimit = 4.0;

$processedThisStep = 0;

try {
    while ($index < $totalEntries) {
        if ($processedThisStep >= $batchLimit) {
            break;
        }

        if ((microtime(true) - $startedAt) >= $timeLimit) {
            break;
        }

        $entry = $entries[$index] ?? null;

        if (!is_array($entry)) {
            $index++;
            continue;
        }

        $rawName = (string)($entry['rawName'] ?? '');
        $isDir = !empty($entry['isDir']);
        $size = (int)($entry['size'] ?? 0);

        $pathParts = sb_disk_unpack_safe_path_parts($rawName);

        if (empty($pathParts)) {
            $index++;
            continue;
        }

        if ($isDir) {
            [, $newFolders] = sb_disk_unpack_ensure_folder_path(
                $context,
                $securityContext,
                $targetFolder,
                $pathParts
            );

            $job['createdFolders'] = (int)($job['createdFolders'] ?? 0) + $newFolders;
            $index++;
            $processedThisStep++;
            continue;
        }

        $fileName = array_pop($pathParts);
        $safeFileName = sb_disk_unpack_sanitize_name((string)$fileName, 'file');

        if (!empty($allowedExtensions)) {
            $fileExt = mb_strtolower(pathinfo($safeFileName, PATHINFO_EXTENSION));

            if (!in_array($fileExt, $allowedExtensions, true)) {
                throw new RuntimeException('EXTENSION_NOT_ALLOWED_IN_ARCHIVE: ' . $safeFileName);
            }
        }

        [$destinationFolder, $newFolders] = sb_disk_unpack_ensure_folder_path(
            $context,
            $securityContext,
            $targetFolder,
            $pathParts
        );

        $job['createdFolders'] = (int)($job['createdFolders'] ?? 0) + $newFolders;

        $safeFileName = sb_disk_unpack_unique_name($destinationFolder, $safeFileName, $securityContext);

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
            'type' => sb_disk_unpack_detect_mime($safeFileName),
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

        $job['extractedFiles'] = (int)($job['extractedFiles'] ?? 0) + 1;
        $job['processedSize'] = (int)($job['processedSize'] ?? 0) + max(0, $size);

        $index++;
        $processedThisStep++;
    }
} finally {
    $zip->close();
}

$job['index'] = $index;

$done = $index >= $totalEntries;
$percent = $totalEntries > 0 ? (int)floor(($index / $totalEntries) * 100) : 100;

if ($done) {
    $percent = 100;
    sb_disk_unpack_delete_job($jobId);
} else {
    sb_disk_unpack_save_job($job);
}

DiskResponse::success([
    'jobId' => $jobId,
    'done' => $done,
    'index' => $index,
    'totalEntries' => $totalEntries,
    'percent' => $percent,

    'targetFolder' => [
        'id' => (int)$targetFolder->getId(),
        'name' => (string)$targetFolder->getName(),
    ],

    'extractedFiles' => (int)($job['extractedFiles'] ?? 0),
    'createdFolders' => (int)($job['createdFolders'] ?? 0),
    'processedSize' => (int)($job['processedSize'] ?? 0),
    'totalSize' => (int)($job['totalSize'] ?? 0),
]);


---

5. В script.js добавь метод пошаговой распаковки

Файл:

/local/sitebuilder/components/disk/script.js

Перед секцией:

/* =========================================================
   EVENTS
   ========================================================= */

вставь:

DiskComponent.prototype.sleep = function (ms) {
  return new Promise(function (resolve) {
    setTimeout(resolve, ms);
  });
};

DiskComponent.prototype.unpackArchiveFromRow = async function (unpackRow) {
  var fileName = unpackRow.getAttribute('data-name') || 'архив';

  this.showUnpackStatusModal(fileName);

  this.updateUnpackStatusModal({
    percent: 1,
    message: 'Создаю задачу распаковки...',
    info: 'Проверяю архив.'
  });

  var startPayload = this.getBasePayload();

  startPayload.fileId = Number(unpackRow.getAttribute('data-id') || 0);
  startPayload.sessid = this.getSessid();

  var startRes = await this.api('unpackArchiveStart', startPayload);

  if (!startRes || !startRes.ok) {
    this.finishUnpackStatusModal(
      false,
      (startRes && (startRes.message || startRes.error)) || 'Не удалось начать распаковку',
      {}
    );
    return;
  }

  var startData = startRes.data || {};
  var jobId = startData.jobId || '';

  if (!jobId) {
    this.finishUnpackStatusModal(false, 'UNPACK_JOB_ID_EMPTY', {});
    return;
  }

  var done = false;
  var lastData = null;

  while (!done) {
    var stepPayload = this.getBasePayload();

    stepPayload.jobId = jobId;
    stepPayload.sessid = this.getSessid();

    var stepRes = await this.api('unpackArchiveStep', stepPayload);

    if (!stepRes || !stepRes.ok) {
      this.finishUnpackStatusModal(
        false,
        (stepRes && (stepRes.message || stepRes.error)) || 'Ошибка шага распаковки',
        {}
      );
      return;
    }

    var stepData = stepRes.data || {};

    lastData = stepData;
    done = !!stepData.done;

    this.updateUnpackStatusModal({
      percent: stepData.percent || 1,
      message: done ? 'Завершаю распаковку...' : 'Распаковываю файлы...',
      info:
        'Обработано: ' +
        (stepData.index || 0) +
        ' из ' +
        (stepData.totalEntries || 0) +
        ' · Файлов: ' +
        (stepData.extractedFiles || 0)
    });

    if (!done) {
      await this.sleep(150);
    }
  }

  this.finishUnpackStatusModal(true, 'Распаковка завершена', {
    extractedFiles: lastData && lastData.extractedFiles ? lastData.extractedFiles : 0,
    createdFolders: lastData && lastData.createdFolders ? lastData.createdFolders : 0,
    totalSize: lastData && lastData.totalSize ? lastData.totalSize : 0
  });

  var targetFolder = lastData && lastData.targetFolder ? lastData.targetFolder : null;

  var self = this;

  setTimeout(async function () {
    if (targetFolder && targetFolder.id) {
      await self.loadFolder(Number(targetFolder.id));
    } else {
      await self.loadFolder(self.state.currentFolderId || self.state.rootFolderId);
    }
  }, 700);
};


---

6. Замени блок unpackBtn

В script.js найди блок:

var unpackBtn = e.target.closest('[data-row-action="unpack"]');

Замени весь блок на:

var unpackBtn = e.target.closest('[data-row-action="unpack"]');

if (unpackBtn) {
  var unpackRow = e.target.closest('[data-id][data-entity-type="file"]');

  if (!unpackRow) {
    return;
  }

  var fileName = unpackRow.getAttribute('data-name') || 'архив';

  var confirmUnpack = window.confirm(
    'Распаковать архив "' + fileName + '"?\n\n' +
    'Файлы будут распакованы в текущую папку.'
  );

  if (!confirmUnpack) {
    return;
  }

  try {
    self.setLoading(true);

    await self.unpackArchiveFromRow(unpackRow);
  } catch (err) {
    console.error(err);

    self.finishUnpackStatusModal(
      false,
      err && err.message ? err.message : 'Ошибка распаковки',
      {}
    );
  } finally {
    self.setLoading(false);
  }

  return;
}


---

7. Старый unpack_archive.php

Старый файл можно оставить:

/local/sitebuilder/components/disk/actions/unpack_archive.php

Он больше не будет использоваться, если JS вызывает:

unpackArchiveStart
unpackArchiveStep


---

После этого сделай Ctrl + F5 и попробуй распаковать тот же ZIP.

Теперь не должно быть 504, потому что архив будет распаковываться маленькими шагами.