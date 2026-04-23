Отлично. Тогда делаем 2 доработки:

1. “Открыть” = просмотр в браузере, а не только скачивание


2. в breadcrumbs скрыть “Общий диск / SiteBuilder”



Ниже даю готовые правки.


---

1. Просмотр файлов прямо в браузере

Самый надежный способ для твоей сборки — не пытаться угадывать внутренние URL Bitrix Disk, а сделать свой preview endpoint, который:

проверяет контекст siteId/pageId/blockId

проверяет права

находит Bitrix\Disk\File

отдает файл с Content-Disposition: inline


Тогда:

pdf, png, jpg, svg, txt, json, html будут открываться в браузере

остальные типы браузер сам либо покажет, либо предложит скачать



---

Создай файл

/local/sitebuilder/components/disk/preview.php

С таким кодом:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once __DIR__ . '/bootstrap.php';

use Bitrix\Disk\File;

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

    $adapter = new DiskBitrixStorageAdapter($currentUserId);
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

    $mimeType = $fileArray['CONTENT_TYPE'] ?? '';
    if ($mimeType === '') {
        $mimeType = 'application/octet-stream';
    }

    $fileName = (string)$file->getName();
    $fileSize = filesize($absolutePath);

    while (ob_get_level()) {
        ob_end_clean();
    }

    header('Content-Type: ' . $mimeType);
    header('Content-Length: ' . (int)$fileSize);
    header('Content-Disposition: inline; filename="' . rawurlencode($fileName) . '"');
    header('X-Content-Type-Options: nosniff');

    readfile($absolutePath);
    exit;
} catch (Throwable $e) {
    http_response_code(500);
    header('Content-Type: text/plain; charset=UTF-8');
    echo $e->getMessage();
    exit;
}


---

Теперь обнови DiskBitrixStorageAdapter.php

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Замени normalizeFile() на это

protected function normalizeFile(DiskContext $context, File $file): array
{
    $name = (string)$file->getName();
    $extension = (string)$file->getExtension();
    $mimeType = $this->detectMimeTypeByExtension($extension);

    return [
        'id' => (int)$file->getId(),
        'entityType' => 'file',
        'name' => $name,
        'originalName' => $name,
        'extension' => $extension,
        'mimeType' => $mimeType,
        'size' => (int)$file->getSize(),
        'downloadUrl' => $this->buildDownloadUrl($file),
        'previewUrl' => $this->buildPreviewUrl($context, $file),
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}

Добавь в класс новый метод

protected function buildPreviewUrl(DiskContext $context, File $file): string
{
    return '/local/sitebuilder/components/disk/preview.php'
        . '?siteId=' . (int)$context->siteId
        . '&pageId=' . (int)$context->pageId
        . '&blockId=' . (int)$context->blockId
        . '&fileId=' . (int)$file->getId();
}


---

Обнови script.js

Файл:

/local/sitebuilder/components/disk/script.js

В renderItemsTable() добавь data-preview-url

Найди строку с data-download-url и добавь сразу после нее:

'data-preview-url="' + escapeHtml(item.previewUrl || '') + '" ' +

То есть кусок должен стать таким:

'<tr class="sb-disk__row ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +
  'data-id="' + escapeHtml(item.id) + '" ' +
  'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
  'data-name="' + escapeHtml(item.name) + '" ' +
  'data-download-url="' + escapeHtml(item.downloadUrl || '') + '" ' +
  'data-preview-url="' + escapeHtml(item.previewUrl || '') + '">' +

В renderItemsGrid() тоже добавь data-preview-url

'data-preview-url="' + escapeHtml(item.previewUrl || '') + '" ' +

Итоговый кусок:

'<div class="sb-disk__card ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +
     'data-id="' + escapeHtml(item.id) + '" ' +
     'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
     'data-name="' + escapeHtml(item.name) + '" ' +
     'data-download-url="' + escapeHtml(item.downloadUrl || '') + '" ' +
     'data-preview-url="' + escapeHtml(item.previewUrl || '') + '">' +

И в обработчике open замени открытие файла

Найди кусок:

} else if (entityType === 'file') {
  var downloadUrl = row.getAttribute('data-download-url') || '';
  if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
}

и замени на:

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

2. Скрыть “Общий диск / SiteBuilder” в breadcrumbs

Сейчас breadcrumbs приходят полные.
Самый удобный вариант — обрезать их на фронте, начиная с rootFolderId.

В script.js замени renderBreadcrumbs()

Найди метод:

DiskComponent.prototype.renderBreadcrumbs = function () {

И замени целиком на это:

DiskComponent.prototype.renderBreadcrumbs = function () {
  var container = this.root.querySelector('[data-role="breadcrumbs"]');
  if (!container) return;

  var crumbs = Array.isArray(this.state.breadcrumbs) ? this.state.breadcrumbs.slice() : [];

  if (this.state.rootFolderId) {
    var startIndex = crumbs.findIndex(function (item) {
      return Number(item.id || 0) === Number(this.state.rootFolderId || 0);
    }, this);

    if (startIndex >= 0) {
      crumbs = crumbs.slice(startIndex);
    }
  }

  if (crumbs.length && this.state.settings && this.state.settings.title) {
    crumbs[0] = {
      id: crumbs[0].id,
      name: this.state.settings.title
    };
  }

  container.innerHTML = crumbs.map(function (item) {
    return '<button type="button" class="sb-disk__crumb" data-folder-id="' + escapeHtml(item.id) + '">' +
      escapeHtml(item.name) +
    '</button>';
  }).join('<span>/</span>');
};

Что это даст

Было:

Общий диск / SiteBuilder / Сайт Тестовый сайт 2 / Папка

Станет:

Файлы / Папка

или:

Документы / Архив

если у блока title = "Документы".


---

Что сделать после правок

1. сохранить preview.php


2. сохранить DiskBitrixStorageAdapter.php


3. сохранить script.js


4. сделать Ctrl + F5




---

Что должно получиться

кнопка Открыть у файла будет открывать файл в браузере

скачивание через bulk action останется отдельным

breadcrumbs станут короче и чище

“Общий диск / SiteBuilder” исчезнут из UI


Если хочешь, следующим сообщением я дам еще и аккуратную визуальную доработку styles.css, чтобы этот блок выглядел уже почти как готовый корпоративный компонент.