Да, так даже удобнее. Делаем так:

Архив лежит в папке “Диск 2”
Нажал “Распаковать”
Файлы появляются прямо в “Диск 2”
Новая папка с названием архива больше НЕ создаётся


---

1. Правим unpack_archive.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive.php

1.1. Найди этот кусок

$currentFolderId = (int)($data['currentFolderId'] ?? 0);

if ($fileId <= 0) {
    throw new RuntimeException('INVALID_FILE_ID');
}

if ($currentFolderId <= 0) {
    throw new RuntimeException('INVALID_FOLDER_ID');
}

DiskValidator::assertFolderInsideRoot($currentFolderId, $rootFolderId, $context);

Замени на:

if ($fileId <= 0) {
    throw new RuntimeException('INVALID_FILE_ID');
}


---

1.2. Найди этот большой блок

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

Замени на:

$securityContext = Driver::getInstance()->getFakeSecurityContext($context->currentUserId);

/*
 * Распаковываем НЕ в новую папку,
 * а прямо туда, где лежит сам архив.
 */
$targetFolder = Folder::loadById($sourceParentId);

if (!$targetFolder instanceof Folder) {
    $zip->close();
    throw new RuntimeException('ARCHIVE_PARENT_FOLDER_NOT_FOUND');
}


---

1.3. Найди:

$createdFolders = 1;

Замени на:

$createdFolders = 0;


---

1.4. В конце ответ можно оставить как есть

Найди:

DiskResponse::success([
    'targetFolder' => [
        'id' => (int)$targetFolder->getId(),
        'name' => (string)$targetFolder->getName(),
    ],
    'extractedFiles' => $extractedFiles,
    'createdFolders' => $createdFolders,
    'totalSize' => $totalSize,
]);

Оставь без изменений.

Теперь после распаковки JS откроет ту же папку, где лежал архив.


---

2. Правим текст в script.js

Файл:

/local/sitebuilder/components/disk/script.js

Найди в обработчике unpack:

var confirmUnpack = window.confirm(
  'Распаковать архив "' + fileName + '"?\n\n' +
  'Будет создана новая папка с содержимым архива.'
);

Замени на:

var confirmUnpack = window.confirm(
  'Распаковать архив "' + fileName + '"?\n\n' +
  'Файлы будут распакованы в текущую папку.'
);


---

2.1. Если добавлял модальное окно распаковки

Найди в showUnpackStatusModal строку:

info.textContent = 'Создаю папку и проверяю архив...';

Замени на:

info.textContent = 'Проверяю архив и подготавливаю распаковку...';


---

3. Что будет с одинаковыми именами

Если в папке уже есть файл с таким именем, код не перезатрёт его, а создаст копию:

document.docx
document (1).docx
document (2).docx

Если в архиве есть папка:

images/photo.jpg

то папка images будет создана прямо рядом с архивом.

Если папка images уже есть, файлы попадут в существующую images.


---

После правок сделай Ctrl + F5 и попробуй распаковать ZIP.