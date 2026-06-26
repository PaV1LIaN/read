Это точно session lock Bitrix.

Ошибка прямо говорит:

Unable to get session lock within 60 seconds

То есть один запрос, скорее всего загрузка файла, держит сессию пользователя. В это время другой запрос /local/sitebuilder/public.php пытается открыть ту же сессию и ждёт 60 секунд, потом падает.

Нужно в upload/распаковке освобождать сессию после проверки прав.


---

1. Добавь helper в components/disk/bootstrap.php

Файл:

/local/sitebuilder/components/disk/bootstrap.php

В самый конец файла добавь:

if (!function_exists('sb_disk_release_session_lock')) {
    function sb_disk_release_session_lock(): void
    {
        /*
         * После проверки авторизации, sessid и прав
         * освобождаем lock PHP/Bitrix-сессии.
         *
         * Иначе большой upload/распаковка держит сессию,
         * а остальные страницы Bitrix в этом же браузере ждут 60 секунд
         * и падают с "Unable to get session lock".
         */
        try {
            if (class_exists('\Bitrix\Main\Application')) {
                $session = \Bitrix\Main\Application::getInstance()->getSession();

                if (method_exists($session, 'save')) {
                    $session->save();
                    return;
                }
            }
        } catch (Throwable $e) {
            // fallback ниже
        }

        if (session_status() === PHP_SESSION_ACTIVE) {
            @session_write_close();
        }
    }
}


---

2. Исправь upload.php

Файл:

/local/sitebuilder/components/disk/actions/upload.php

Найди место после проверки прав:

DiskValidator::assertCan($permissions, 'canView');
DiskValidator::assertCan($permissions, 'canUpload');

Сразу после этого вставь:

sb_disk_release_session_lock();

Должно быть так:

DiskValidator::assertCan($permissions, 'canView');
DiskValidator::assertCan($permissions, 'canUpload');

sb_disk_release_session_lock();

Важно: вставлять после DiskCsrf::validateFromRequest(), DiskCurrentUser::requireId() и проверок прав, но до обработки $_FILES и загрузки в Диск.


---

3. Исправь распаковку

unpack_archive_start.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_start.php

После проверок доступа и проверки архива вставь:

sb_disk_release_session_lock();

Лучшее место — после этого блока:

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

Сразу после него:

sb_disk_release_session_lock();


---

unpack_archive_step.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_step.php

Найди место после загрузки $file и $targetFolder.

После этого блока:

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

Вставь:

sb_disk_release_session_lock();


---

4. Проверь PHP upload progress

На сервере выполни:

php -i | grep session.upload_progress

Если увидишь:

session.upload_progress.enabled => On

лучше выключить в php.ini:

session.upload_progress.enabled = Off

Потом перезапусти PHP-FPM:

sudo systemctl restart php8.1-fpm

Если версия PHP другая — замени 8.1 на свою.


---

5. Почему это происходит

Схема такая:

1. Ты начал загрузку большого файла.
2. Bitrix открыл PHP-сессию пользователя.
3. Пока upload-запрос работает, сессия заблокирована.
4. Ты открываешь другую страницу Bitrix.
5. Она тоже хочет открыть эту же сессию.
6. Ждёт 60 секунд.
7. Получаешь:
   Unable to get session lock within 60 seconds.

После sb_disk_release_session_lock() upload будет продолжаться, но остальные страницы Bitrix смогут открываться параллельно.

После правки сделай Ctrl + F5, начни загрузку большого файла и параллельно попробуй открыть любую страницу Bitrix.