Да, для распаковки это тоже работает. Даже лучше, чем для загрузки, потому что распаковка у нас уже пошаговая:

unpackArchiveStart  → проверяет архив и создаёт задачу
unpackArchiveStep   → распаковывает по 1 файлу/папке за запрос

Чтобы Bitrix не блокировал сессию, нужно закрывать session lock после проверки авторизации/прав, но до работы с ZIP.


---

Куда ставить

1. unpack_archive_start.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_start.php

После проверок:

DiskValidator::assertCan($permissions, 'canView');
DiskValidator::assertCan($permissions, 'canUpload');

и после проверки, что архив лежит внутри нужного root, вставь:

sb_disk_release_session_lock();

Нормальное место вот здесь:

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

/*
 * Освобождаем сессию до чтения ZIP.
 */
sb_disk_release_session_lock();


---

2. unpack_archive_step.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_step.php

После загрузки файла и папки:

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

/*
 * Освобождаем сессию до чтения ZIP и загрузки файлов в Disk.
 */
sb_disk_release_session_lock();


---

Важно

Не ставь sb_disk_release_session_lock() в самое начало файла.

Нельзя делать так:

sb_disk_release_session_lock();

DiskCsrf::validateFromRequest();
$currentUserId = DiskCurrentUser::requireId();

Правильно только так:

DiskCsrf::validateFromRequest();
$currentUserId = DiskCurrentUser::requireId();

/* проверили контекст, root, права */

sb_disk_release_session_lock();

/* тяжёлая работа */


---

Если старый unpackArchive ещё используется

Если где-то ещё вызывается старое действие:

unpackArchive

то в файл тоже добавь:

/local/sitebuilder/components/disk/actions/unpack_archive.php

после проверки прав:

DiskValidator::assertCan($permissions, 'canView');
DiskValidator::assertCan($permissions, 'canUpload');

sb_disk_release_session_lock();

Но если JS уже вызывает только:

unpackArchiveStart
unpackArchiveStep

то старый файл можно не трогать.


---

Итог: да, для распаковки работает. Для твоей пошаговой распаковки нужно поставить sb_disk_release_session_lock() в unpack_archive_start.php и unpack_archive_step.php.