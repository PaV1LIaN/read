Да, ты прав. Я неточно сказал: теоретически jobId позволяет несколько задач, но в твоей текущей связке один пользователь не может нормально запустить вторую распаковку, потому что второй запрос упирается в Bitrix/PHP-сессию или ошибку backend, и вместо JSON прилетает HTML.

Нормальное поведение должно быть такое:

1. Пользователь уже распаковывает архив.
2. Он нажимает распаковать второй.
3. Сервер НЕ падает HTML.
4. Сервер отдаёт JSON:
   "У вас уже идёт распаковка архива. Дождитесь завершения."

Сделаем правильно: серверный lock + всегда JSON-ответ.


---

1. Сначала исправь api.php, чтобы bootstrap тоже был внутри try

Сейчас у тебя, скорее всего, так:

require_once __DIR__ . '/bootstrap.php';

try {
    switch ($action) {
        ...
    }
} catch (Throwable $e) {
    ...
}

Это плохо, потому что если ошибка происходит внутри bootstrap.php, Bitrix может вернуть HTML.

Нужно сделать так:

try {
    require_once __DIR__ . '/bootstrap.php';

    if (in_array($action, ['list', 'search', 'bootstrap'], true)) {
        ob_start('sb_disk_enrich_json_response_with_user_names');
    }

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

        case 'unpackArchiveStart':
            require __DIR__ . '/actions/unpack_archive_start.php';
            break;

        case 'unpackArchiveStep':
            require __DIR__ . '/actions/unpack_archive_step.php';
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
    while (ob_get_level() > 0) {
        @ob_end_clean();
    }

    if (class_exists('DiskResponse')) {
        DiskResponse::error('SERVER_ERROR', $e->getMessage(), [
            'file' => $e->getFile(),
            'line' => $e->getLine(),
        ]);
    }

    sb_disk_force_json_error(
        'SERVER_ERROR',
        $e->getMessage(),
        [
            'file' => $e->getFile(),
            'line' => $e->getLine(),
        ]
    );
}

То есть главное: require_once __DIR__ . '/bootstrap.php'; должен быть внутри try.


---

2. Добавь lock-функции в unpack_archive_lib.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_lib.php

В конец файла добавь:

if (!function_exists('sb_disk_unpack_locks_dir')) {
    function sb_disk_unpack_locks_dir(): string
    {
        $dir = rtrim((string)$_SERVER['DOCUMENT_ROOT'], '/') . '/upload/sitebuilder/disk_unpack_locks';

        if (!is_dir($dir)) {
            mkdir($dir, 0775, true);
        }

        return $dir;
    }
}

if (!function_exists('sb_disk_unpack_lock_path')) {
    function sb_disk_unpack_lock_path(string $type, int $id): string
    {
        $type = preg_replace('/[^a-z0-9_]/i', '', $type);

        if ($type === '') {
            throw new RuntimeException('INVALID_LOCK_TYPE');
        }

        if ($id <= 0) {
            throw new RuntimeException('INVALID_LOCK_ID');
        }

        return sb_disk_unpack_locks_dir() . '/' . $type . '_' . $id . '.lock';
    }
}

if (!function_exists('sb_disk_unpack_is_job_active')) {
    function sb_disk_unpack_is_job_active(string $jobId): bool
    {
        $jobId = preg_replace('/[^a-f0-9]/i', '', $jobId);

        if ($jobId === '') {
            return false;
        }

        try {
            $job = sb_disk_unpack_load_job($jobId);
        } catch (Throwable $e) {
            return false;
        }

        $updatedAt = (int)($job['updatedAt'] ?? $job['createdAt'] ?? 0);

        if ($updatedAt <= 0) {
            return false;
        }

        /*
         * Если задача не обновлялась больше часа,
         * считаем lock зависшим.
         */
        return (time() - $updatedAt) < 3600;
    }
}

if (!function_exists('sb_disk_unpack_cleanup_lock')) {
    function sb_disk_unpack_cleanup_lock(string $type, int $id): void
    {
        $path = sb_disk_unpack_lock_path($type, $id);

        if (!is_file($path)) {
            return;
        }

        $data = json_decode((string)file_get_contents($path), true);

        if (!is_array($data)) {
            @unlink($path);
            return;
        }

        $jobId = (string)($data['jobId'] ?? '');

        if ($jobId === '' || !sb_disk_unpack_is_job_active($jobId)) {
            @unlink($path);
        }
    }
}

if (!function_exists('sb_disk_unpack_acquire_lock')) {
    function sb_disk_unpack_acquire_lock(string $type, int $id, string $jobId, string $message): void
    {
        sb_disk_unpack_cleanup_lock($type, $id);

        $path = sb_disk_unpack_lock_path($type, $id);

        $handle = @fopen($path, 'x');

        if (!is_resource($handle)) {
            throw new RuntimeException($message);
        }

        fwrite($handle, json_encode([
            'type' => $type,
            'id' => $id,
            'jobId' => $jobId,
            'createdAt' => time(),
        ], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES));

        fclose($handle);
    }
}

if (!function_exists('sb_disk_unpack_release_lock')) {
    function sb_disk_unpack_release_lock(string $type, int $id, string $jobId = ''): void
    {
        $path = sb_disk_unpack_lock_path($type, $id);

        if (!is_file($path)) {
            return;
        }

        if ($jobId !== '') {
            $data = json_decode((string)file_get_contents($path), true);

            if (is_array($data) && (string)($data['jobId'] ?? '') !== $jobId) {
                return;
            }
        }

        @unlink($path);
    }
}

if (!function_exists('sb_disk_unpack_release_job_locks')) {
    function sb_disk_unpack_release_job_locks(array $job): void
    {
        $jobId = (string)($job['id'] ?? '');
        $userId = (int)($job['currentUserId'] ?? 0);
        $targetFolderId = (int)($job['targetFolderId'] ?? 0);

        if ($userId > 0) {
            sb_disk_unpack_release_lock('user', $userId, $jobId);
        }

        if ($targetFolderId > 0) {
            sb_disk_unpack_release_lock('folder', $targetFolderId, $jobId);
        }
    }
}

if (!function_exists('sb_disk_unpack_touch_job_locks')) {
    function sb_disk_unpack_touch_job_locks(array $job): void
    {
        $userId = (int)($job['currentUserId'] ?? 0);
        $targetFolderId = (int)($job['targetFolderId'] ?? 0);

        if ($userId > 0) {
            $path = sb_disk_unpack_lock_path('user', $userId);

            if (is_file($path)) {
                @touch($path);
            }
        }

        if ($targetFolderId > 0) {
            $path = sb_disk_unpack_lock_path('folder', $targetFolderId);

            if (is_file($path)) {
                @touch($path);
            }
        }
    }
}


---

3. Обнови сохранение job

В этом же файле найди функцию:

function sb_disk_unpack_save_job(array $job): void

Внутри перед file_put_contents добавь:

$job['updatedAt'] = time();

Должно быть так:

function sb_disk_unpack_save_job(array $job): void
{
    $jobId = (string)($job['id'] ?? '');

    if ($jobId === '') {
        throw new RuntimeException('EMPTY_UNPACK_JOB_ID');
    }

    $job['updatedAt'] = time();

    file_put_contents(
        sb_disk_unpack_job_path($jobId),
        json_encode($job, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES)
    );
}


---

4. Правим unpack_archive_start.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_start.php

Найди место, где создаётся $jobId:

$jobId = bin2hex(random_bytes(16));

После формирования массива $job = [...], но до:

sb_disk_unpack_save_job($job);

вставь:

$userLockAcquired = false;
$folderLockAcquired = false;

try {
    /*
     * Один пользователь — одна активная распаковка.
     */
    sb_disk_unpack_acquire_lock(
        'user',
        $context->currentUserId,
        $jobId,
        'У вас уже идёт распаковка архива. Дождитесь завершения.'
    );

    $userLockAcquired = true;

    /*
     * В одну папку нельзя распаковывать два архива одновременно.
     */
    sb_disk_unpack_acquire_lock(
        'folder',
        (int)$targetFolder->getId(),
        $jobId,
        'В этой папке уже идёт распаковка архива. Дождитесь завершения.'
    );

    $folderLockAcquired = true;
} catch (Throwable $e) {
    if ($userLockAcquired) {
        sb_disk_unpack_release_lock('user', $context->currentUserId, $jobId);
    }

    if ($folderLockAcquired) {
        sb_disk_unpack_release_lock('folder', (int)$targetFolder->getId(), $jobId);
    }

    throw $e;
}

И в сам $job добавь:

'updatedAt' => time(),

Например:

$job = [
    'id' => $jobId,
    'createdAt' => time(),
    'updatedAt' => time(),

    'siteId' => $context->siteId,
    ...
];


---

5. Правим unpack_archive_step.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_step.php

Найди в конце:

if ($done) {
    $percent = 100;
    sb_disk_unpack_delete_job($jobId);
} else {
    sb_disk_unpack_save_job($job);
}

Замени на:

if ($done) {
    $percent = 100;

    sb_disk_unpack_release_job_locks($job);
    sb_disk_unpack_delete_job($jobId);
} else {
    sb_disk_unpack_touch_job_locks($job);
    sb_disk_unpack_save_job($job);
}


---

Что получится после правки

Один пользователь запускает первый архив

ОК, задача создана.

Он же пытается запустить второй архив

У вас уже идёт распаковка архива. Дождитесь завершения.

И это будет JSON, не HTML.

Другой пользователь пытается распаковать архив в другую папку

ОК, можно.

Другой пользователь пытается распаковать архив в ту же папку

В этой папке уже идёт распаковка архива. Дождитесь завершения.


---

Главное

Сейчас ошибка “ожидался JSON, пришёл HTML” — это не нормальный запрет второй распаковки. Это аварийное поведение Bitrix/API.

Нормальный запрет должен быть через lock и JSON-ответ.