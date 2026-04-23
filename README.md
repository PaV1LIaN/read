Да, это лучше делать не через наш preview.php, а через штатный обработчик документов Bitrix Disk, если у вас уже настроен Office Online / Microsoft Office Online. Bitrix24 Drive умеет работать с документами через встроенный редактор Bitrix24.Docs, Office 365 / MS Office Online и Google Docs, а доступные действия зависят от выбранного способа работы с документами и настроенной интеграции. 

С учетом того, что у тебя уже были логи про WOPI discovery и document handlers, правильная схема такая:

для pdf/png/jpg/txt/json/html оставляем наш preview.php

для doc/docx/xls/xlsx/ppt/pptx уводим в Bitrix document viewer/editor

кнопки делаем две:

Открыть → браузерный preview или Office Online

Скачать → прямое скачивание



Что менять

1. В DiskBitrixStorageAdapter.php добавь определение office-файлов

Ниже detectMimeTypeByExtension() добавь:

protected function isOfficeDocument(string $extension): bool
{
    $extension = mb_strtolower(trim($extension));

    return in_array($extension, [
        'doc', 'docx',
        'xls', 'xlsx',
        'ppt', 'pptx',
        'odt', 'ods', 'odp',
        'rtf', 'csv'
    ], true);
}


---

2. Замени normalizeFile() на этот вариант

protected function normalizeFile(DiskContext $context, File $file): array
{
    $name = (string)$file->getName();
    $extension = (string)$file->getExtension();
    $mimeType = $this->detectMimeTypeByExtension($extension);

    $previewUrl = $this->isOfficeDocument($extension)
        ? $this->buildOfficePreviewUrl($context, $file)
        : $this->buildPreviewUrl($context, $file);

    return [
        'id' => (int)$file->getId(),
        'entityType' => 'file',
        'name' => $name,
        'originalName' => $name,
        'extension' => $extension,
        'mimeType' => $mimeType,
        'size' => (int)$file->getSize(),
        'downloadUrl' => $this->buildDownloadUrl($file),
        'previewUrl' => $previewUrl,
        'previewMode' => $this->isOfficeDocument($extension) ? 'office' : 'browser',
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}


---

3. Добавь в этот же класс метод URL для Office Online

protected function buildOfficePreviewUrl(DiskContext $context, File $file): string
{
    return '/local/sitebuilder/components/disk/office_preview.php'
        . '?siteId=' . (int)$context->siteId
        . '&pageId=' . (int)$context->pageId
        . '&blockId=' . (int)$context->blockId
        . '&fileId=' . (int)$file->getId();
}


---

4. Создай файл office_preview.php

Путь:

/local/sitebuilder/components/disk/office_preview.php

Код:

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

    $file = File::loadById($fileId);
    if (!$file instanceof File) {
        throw new RuntimeException('DISK_FILE_NOT_FOUND');
    }

    // Универсальный безопасный вариант:
    // открываем стандартную страницу Disk, а Bitrix сам решит,
    // показать preview, Office Online или скачать файл.
    $url = '/company/personal/user/' . $currentUserId . '/disk/path/'
        . rawurlencode($file->getName())
        . '?objectId=' . (int)$file->getId()
        . '&IFRAME=Y';

    LocalRedirect($url);
} catch (Throwable $e) {
    http_response_code(500);
    header('Content-Type: text/plain; charset=UTF-8');
    echo $e->getMessage();
    exit;
}


---

Важно

Этот редирект — самый безопасный интеграционный вариант, потому что точные внутренние методы Bitrix для Office Online сильно зависят от версии и установленного document handler. А стандартный интерфейс Disk уже знает, когда открывать документ через настроенный сервис. Возможность работать с документами через MS Office Online / Office 365 / Google Docs в Bitrix24 предусмотрена, но конкретный runtime-маршрут зависит от вашей конфигурации. 


---

5. В script.js ничего большого менять не надо

У тебя уже есть логика:

if (previewUrl) {
  window.open(previewUrl, '_blank');
} else if (downloadUrl) {
  window.open(downloadUrl, '_blank');
}

Этого достаточно.


---

По breadcrumbs

Ты просил скрыть Общий диск / SiteBuilder.
Мы уже резали breadcrumbs от rootFolderId, но если первый breadcrumb все равно не нравится, оставь такой метод в script.js:

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

  if (crumbs.length) {
    crumbs[0] = {
      id: crumbs[0].id,
      name: this.state.settings && this.state.settings.title ? this.state.settings.title : 'Файлы'
    };
  }

  container.innerHTML = crumbs.map(function (item) {
    return '<button type="button" class="sb-disk__crumb" data-folder-id="' + escapeHtml(item.id) + '">' +
      escapeHtml(item.name) +
    '</button>';
  }).join('<span>/</span>');
};

Тогда будет что-то вроде:

Файлы / Папка / Архив

а не Общий диск / SiteBuilder / ...


---

Что важно проверить

После этого проверь на одном docx:

если настроен document handler и WOPI живой, файл должен уйти в штатный viewer/editor Bitrix

если handler не работает, откроется стандартная страница Disk или пойдет fallback на скачивание


С учетом твоей прошлой ошибки с expired certificate на hosting/discovery, Office Online может не заработать, пока не починен сертификат у WOPI discovery. Это уже не проблема компонента, а инфраструктуры document handler. Если хочешь, следующим сообщением я дам готовый DiskBitrixStorageAdapter.php целиком с этими изменениями.