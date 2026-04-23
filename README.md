Проблема уже не в script.js.
По твоему public_page.php видно, что ты все подключаешь правильно для обычных расширений, но BX.Disk.Viewer на этой кастомной странице все равно не появляется.

Значит для этого layout путь через встроенный viewer на самой странице ненадежен.

Практичное решение

Сделать так:

Открыть office-файл не внутри текущей страницы

а через штатную страницу Disk / viewer Bitrix

Редактировать тоже открывать через штатную страницу Bitrix Disk с action edit


То есть не пытаться поднимать Disk Viewer внутри sitebuilder public_page.php, а делегировать это стандартному интерфейсу Bitrix, где viewer уже точно живет.

Это самый стабильный вариант для твоей сборки.


---

Что сделать

1. Обновить DiskBitrixStorageAdapter.php

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Замени normalizeFile() на это:

protected function normalizeFile(DiskContext $context, File $file): array
{
    $name = (string)$file->getName();
    $extension = (string)$file->getExtension();
    $mimeType = $this->detectMimeTypeByExtension($extension);

    $isOffice = $this->isOfficeDocument($extension);

    return [
        'id' => (int)$file->getId(),
        'entityType' => 'file',
        'name' => $name,
        'originalName' => $name,
        'extension' => $extension,
        'mimeType' => $mimeType,
        'size' => (int)$file->getSize(),
        'downloadUrl' => $this->buildDownloadUrl($file),
        'previewUrl' => $isOffice
            ? $this->buildDiskOpenUrl($file)
            : $this->buildPreviewUrl($context, $file),
        'editUrl' => $isOffice
            ? $this->buildDiskEditUrl($file)
            : '',
        'previewMode' => $isOffice ? 'office' : 'browser',
        'canEdit' => $isOffice,
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}

Добавь в этот класс методы:

protected function buildDiskOpenUrl(File $file): string
{
    return '/bitrix/components/bitrix/disk.file.view/templates/.default/show_file.php'
        . '?objectId=' . (int)$file->getId()
        . '&service=driver'
        . '&showInline=1'
        . '&ncc=1';
}

protected function buildDiskEditUrl(File $file): string
{
    return '/bitrix/components/bitrix/disk.file.view/templates/.default/show_file.php'
        . '?objectId=' . (int)$file->getId()
        . '&service=driver'
        . '&action=edit'
        . '&ncc=1';
}


---

2. Обновить script.js

Файл:

/local/sitebuilder/components/disk/script.js

В renderItemsTable() добавь data-edit-url

Найди строку с data-preview-mode и добавь сразу после нее:

'data-edit-url="' + escapeHtml(item.editUrl || '') + '" '

То есть кусок должен стать таким:

'<tr class="sb-disk__row ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +
  'data-id="' + escapeHtml(item.id) + '" ' +
  'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
  'data-name="' + escapeHtml(item.name) + '" ' +
  'data-download-url="' + escapeHtml(item.downloadUrl || '') + '" ' +
  'data-preview-url="' + escapeHtml(item.previewUrl || '') + '" ' +
  'data-preview-mode="' + escapeHtml(item.previewMode || '') + '" ' +
  'data-edit-url="' + escapeHtml(item.editUrl || '') + '">' +

В renderItemsGrid() тоже добавь:

'data-edit-url="' + escapeHtml(item.editUrl || '') + '" '


---

В renderItemsTable() office-кнопки упрости

Для office-файлов вместо span data-viewer ... сделай обычные кнопки.

Заменяй логику openControl/editControl на такую:

var openControl = '';
var editControl = '';

if (item.entityType === 'folder') {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
} else if (item.previewMode === 'office') {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
  editControl = '<button type="button" class="sb-disk__row-btn" data-row-action="edit-office">Редактировать</button>';
} else {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
}

То же самое сделай в renderItemsGrid().


---

В обработчике open замени блок для файла

Найди:

if (previewMode === 'office') {
  return;
}

и замени на:

if (previewMode === 'office') {
  if (previewUrl) {
    window.open(previewUrl, '_blank');
  } else if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
  return;
}


---

В обработчике edit-office замени на:

var editOfficeBtn = e.target.closest('[data-row-action="edit-office"]');
if (editOfficeBtn) {
  var editRow = e.target.closest('[data-id][data-entity-type="file"]');
  if (!editRow) return;

  var editUrl = editRow.getAttribute('data-edit-url') || '';
  var previewModeForEdit = editRow.getAttribute('data-preview-mode') || '';

  if (previewModeForEdit === 'office') {
    if (editUrl) {
      window.open(editUrl, '_blank');
    } else {
      var fallbackPreviewUrl = editRow.getAttribute('data-preview-url') || '';
      if (fallbackPreviewUrl) {
        window.open(fallbackPreviewUrl, '_blank');
      }
    }
  }
  return;
}


---

Почему это лучше

Потому что сейчас на кастомной странице у тебя не поднимается BX.Disk.Viewer, а вот стандартные bitrix-страницы просмотра/редактирования Disk уже умеют:

Office Online

Bitrix24 Docs

document handler

edit mode


То есть мы не ломаемся о кастомный layout и не зависим от отсутствующего BX.Disk.Viewer.


---

Что получится

Открыть у docx/xlsx/pptx → откроет штатный просмотр Bitrix

Редактировать → откроет штатное редактирование Bitrix

картинки/pdf/txt останутся через обычный browser preview

Скачать останется отдельной кнопкой



---

Важно

Если даже на штатной странице Bitrix редактирование не откроется, значит уже проблема в:

document handler

WOPI

сертификате

настройке Office Online / Bitrix Docs


Но это уже будет проблема инфраструктуры, а не компонента.

Если хочешь, я следующим сообщением пришлю тебе полный готовый script.js уже под эту схему без viewer на кастомной странице.