Да, сейчас у тебя логика все еще смешана: кнопка «Открыть» фактически ведет на скачивание.
Правильнее сделать так:

Открыть → просмотр в браузере

Скачать → принудительное скачивание


И при этом понимать, что не все типы файлов браузер умеет показывать. Обычно в браузере откроются:

pdf

png, jpg, jpeg, gif, webp, svg

txt, json, xml, html


А, например, docx/xlsx чаще всего будут скачиваться даже через preview.


---

Что нужно сделать

1. Исправить preview.php

Замени файл:

/local/sitebuilder/components/disk/preview.php

на этот код:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once __DIR__ . '/bootstrap.php';

use Bitrix\Disk\File;

function sb_disk_detect_mime(string $path, string $fileName = ''): string
{
    if (is_file($path) && function_exists('mime_content_type')) {
        $mime = @mime_content_type($path);
        if (is_string($mime) && $mime !== '') {
            return $mime;
        }
    }

    $ext = mb_strtolower(pathinfo($fileName, PATHINFO_EXTENSION));

    $map = [
        'pdf' => 'application/pdf',
        'png' => 'image/png',
        'jpg' => 'image/jpeg',
        'jpeg' => 'image/jpeg',
        'gif' => 'image/gif',
        'webp' => 'image/webp',
        'svg' => 'image/svg+xml',
        'txt' => 'text/plain; charset=UTF-8',
        'json' => 'application/json; charset=UTF-8',
        'xml' => 'application/xml; charset=UTF-8',
        'html' => 'text/html; charset=UTF-8',
        'htm' => 'text/html; charset=UTF-8',
        'csv' => 'text/csv; charset=UTF-8',
        'mp4' => 'video/mp4',
        'mp3' => 'audio/mpeg',
    ];

    return $map[$ext] ?? 'application/octet-stream';
}

try {
    $siteId = (int)($_GET['siteId'] ?? 0);
    $pageId = (int)($_GET['pageId'] ?? 0);
    $blockId = (int)($_GET['blockId'] ?? 0);
    $fileId = (int)($_GET['fileId'] ?? 0);

    $currentUserId = DiskCurrentUser::requireId();

    $context = DiskContextFactory::fromArray([
        'siteId' => $siteId,
        'pageId' => $pageId,
        'blockId' => $blockId,
        'currentUserId' => $currentUserId,
    ]);

    DiskValidator::assertContext($context);

    $settings = DiskSettingsRepository::ensureExistsForBlock(
        $context->blockId,
        $context->siteId,
        $context->pageId,
        $context->currentUserId
    );

    $rootInfo = DiskRootResolver::resolveWithSource($context, $settings, false);
    $permissions = DiskPermissionService::resolve($context, $settings, $rootInfo['rootFolderId']);

    if (empty($permissions['canView'])) {
        throw new RuntimeException('ACCESS_DENIED');
    }

    if ($fileId <= 0) {
        throw new RuntimeException('INVALID_FILE_ID');
    }

    $file = File::loadById($fileId);
    if (!$file instanceof File) {
        throw new RuntimeException('DISK_FILE_NOT_FOUND');
    }

    $fileArray = $file->getFile();
    if (!is_array($fileArray) || empty($fileArray['SRC'])) {
        throw new RuntimeException('FILE_SOURCE_NOT_FOUND');
    }

    $absolutePath = $_SERVER['DOCUMENT_ROOT'] . $fileArray['SRC'];
    if (!is_file($absolutePath)) {
        throw new RuntimeException('FILE_NOT_FOUND_ON_DISK');
    }

    $fileName = (string)$file->getName();
    $mimeType = sb_disk_detect_mime($absolutePath, $fileName);
    $fileSize = (int)filesize($absolutePath);

    while (ob_get_level()) {
        ob_end_clean();
    }

    header('Content-Type: ' . $mimeType);
    header('Content-Length: ' . $fileSize);
    header('Content-Disposition: inline; filename="' . addslashes($fileName) . '"');
    header('X-Content-Type-Options: nosniff');
    header('Cache-Control: private, max-age=0, must-revalidate');

    readfile($absolutePath);
    exit;
} catch (Throwable $e) {
    http_response_code(500);
    header('Content-Type: text/plain; charset=UTF-8');
    echo $e->getMessage();
    exit;
}


---

2. В script.js сделать отдельные кнопки Открыть и Скачать

Сейчас у тебя отдельной кнопки скачивания у строки нет. Добавим.

Файл:

/local/sitebuilder/components/disk/script.js

В renderItemsTable() замени блок actions на этот:

'<td>' +
  '<div class="sb-disk__actions">' +
    '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>' +
    (item.entityType === 'file'
      ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
      : '') +
    '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +
    '<button type="button" class="sb-disk__row-btn" data-row-action="delete">Удалить</button>' +
  '</div>' +
'</td>'

В renderItemsGrid() замени actions на этот:

'<div class="sb-disk__card-actions">' +
  '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>' +
  (item.entityType === 'file'
    ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
    : '') +
  '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +
  '<button type="button" class="sb-disk__row-btn" data-row-action="delete">Удалить</button>' +
'</div>'


---

3. Добавить обработчик download

В bindStaticEvents() внутри обработчика this.root.addEventListener('click', async function (e) { ... })

после блока openBtn вставь:

var downloadBtn = e.target.closest('[data-row-action="download"]');
if (downloadBtn) {
  var downloadRow = e.target.closest('[data-id][data-entity-type="file"]');
  if (!downloadRow) return;

  var directDownloadUrl = downloadRow.getAttribute('data-download-url') || '';
  if (directDownloadUrl) {
    window.open(directDownloadUrl, '_blank');
  }
  return;
}


---

4. Логика кнопки Открыть должна остаться такой

Проверь, чтобы у тебя было именно так:

} else if (entityType === 'file') {
  var previewUrl = row.getAttribute('data-preview-url') || '';
  var downloadUrl = row.getAttribute('data-download-url') || '';

  if (previewUrl) {
    window.open(previewUrl, '_blank');
  } else if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
}


---

Что получится

Теперь поведение будет таким:

Открыть
пытается показать файл в браузере через preview.php

Скачать
всегда идет по downloadUrl



---

Важный момент

Даже после этого некоторые форматы все равно будут скачиваться при “Открыть”, потому что это зависит от браузера и типа файла.

Нормально откроются:

pdf

картинки

txt/json/xml/html


А docx/xlsx чаще всего все равно уйдут в скачивание — это нормально.


---

Если хочешь совсем удобно

Следующим шагом лучше сделать:

для картинок — модальное превью прямо на странице

для PDF — открытие во встроенном iframe/modal

для офисных документов — отдельную кнопку “Скачать”


Тогда UX будет намного лучше.