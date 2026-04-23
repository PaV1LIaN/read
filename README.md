Да, для docx/xlsx/pptx лучше не открывать наш preview.php, а использовать штатный viewer Bitrix, как в твоем примере с data-viewer.

Ниже даю рабочую схему по аналогии с твоим кодом.


---

1. Обнови DiskBitrixStorageAdapter.php

Файл:

/local/sitebuilder/components/disk/lib/DiskBitrixStorageAdapter.php

Замени normalizeFile() на это

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
            ? $this->buildViewerSrc($file)
            : $this->buildPreviewUrl($context, $file),
        'previewMode' => $isOffice ? 'office' : 'browser',
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}

Добавь в класс эти методы

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

protected function buildViewerSrc(File $file): string
{
    $name = (string)$file->getName();
    return '/disk/downloadFile/' . (int)$file->getId() . '/?&ncc=1&filename=' . rawurlencode($name);
}


---

2. Подключи viewer на публичной странице

Файл:

/local/sitebuilder/views/layout/public_page.php

Перед </body> внутри блока if ($pageHasDiskBlock) добавь еще:

<?php
if ($pageHasDiskBlock) {
    CJSCore::Init(['ui.viewer']);
}
?>

Если хочешь, можно вставить прямо перед подключением script.js.

Итоговый блок внизу страницы пусть будет таким:

<?php if ($pageHasDiskBlock): ?>
<?php CJSCore::Init(['ui.viewer']); ?>
<script src="/bitrix/js/main/core/core.js"></script>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js"></script>
<?php endif; ?>


---

3. Обнови script.js

Файл:

/local/sitebuilder/components/disk/script.js

3.1. Добавь в класс метод инициализации viewer

Вставь в файл, например после renderAll():

DiskComponent.prototype.bindViewer = function () {
  if (!window.BX || !BX.UI || !BX.UI.Viewer || !BX.UI.Viewer.Instance) {
    return;
  }

  try {
    BX.UI.Viewer.Instance.bind(this.root);
  } catch (e) {
    console.warn('Viewer bind failed', e);
  }
};


---

3.2. Обнови renderAll()

Найди:

DiskComponent.prototype.renderAll = function () {
  this.renderSubtitle();
  this.renderBreadcrumbs();
  this.renderItemsTable();
  this.renderItemsGrid();
  this.syncSelectedState();
};

Замени на:

DiskComponent.prototype.renderAll = function () {
  this.renderSubtitle();
  this.renderBreadcrumbs();
  this.renderItemsTable();
  this.renderItemsGrid();
  this.syncSelectedState();
  this.bindViewer();
};


---

3.3. Обнови renderItemsTable()

Замени метод целиком на этот:

DiskComponent.prototype.renderItemsTable = function () {
  var tbody = this.root.querySelector('[data-role="items-table"]');
  if (!tbody) return;

  tbody.innerHTML = this.state.items.map(function (item) {
    var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
    var sizeText = item.size ? formatBytes(item.size) : '';
    var badge = item.entityType === 'folder'
      ? '<span class="sb-disk__badge">Папка</span>'
      : '<span class="sb-disk__badge">' + escapeHtml(item.extension || 'Файл') + '</span>';

    var openControl = '';

    if (item.entityType === 'folder') {
      openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
    } else if (item.previewMode === 'office') {
      var actions = escapeHtml(JSON.stringify([{ type: 'download' }]));

      openControl =
        '<span ' +
          'class="sb-disk__row-btn sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
          'data-viewer="" ' +
          'data-viewer-type="cloud-document" ' +
          'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
          'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
          'data-viewer-extension="disk.viewer.document-item" ' +
          'data-object-id="' + escapeHtml(item.id) + '" ' +
          'data-title="' + escapeHtml(item.name) + '" ' +
          'data-actions="' + actions + '"' +
        '>Открыть</span>';
    } else {
      openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
    }

    return '' +
      '<tr class="sb-disk__row ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +
        'data-id="' + escapeHtml(item.id) + '" ' +
        'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
        'data-name="' + escapeHtml(item.name) + '" ' +
        'data-download-url="' + escapeHtml(item.downloadUrl || '') + '" ' +
        'data-preview-url="' + escapeHtml(item.previewUrl || '') + '" ' +
        'data-preview-mode="' + escapeHtml(item.previewMode || '') + '">' +
          '<td>' +
            '<input type="checkbox" class="sb-disk__item-check" data-id="' + escapeHtml(item.id) + '">' +
          '</td>' +
          '<td>' +
            '<div class="sb-disk__item-name">' +
              badge +
              '<span class="sb-disk__item-name-label">' + escapeHtml(item.name) + '</span>' +
            '</div>' +
          '</td>' +
          '<td>' + escapeHtml(typeText) + '</td>' +
          '<td>' + escapeHtml(sizeText) + '</td>' +
          '<td>' + escapeHtml(item.updatedAt || '') + '</td>' +
          '<td>' +
            '<div class="sb-disk__actions">' +
              openControl +
              (item.entityType === 'file'
                ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
                : '') +
              '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="delete">Удалить</button>' +
            '</div>' +
          '</td>' +
      '</tr>';
  }).join('');
};


---

3.4. Обнови renderItemsGrid()

Замени метод целиком на этот:

DiskComponent.prototype.renderItemsGrid = function () {
  var container = this.root.querySelector('[data-view-container="grid"]');
  if (!container) return;

  container.classList.add('sb-disk__grid');

  container.innerHTML = this.state.items.map(function (item) {
    var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
    var sizeText = item.size ? formatBytes(item.size) : '—';

    var openControl = '';

    if (item.entityType === 'folder') {
      openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
    } else if (item.previewMode === 'office') {
      var actions = escapeHtml(JSON.stringify([{ type: 'download' }]));

      openControl =
        '<span ' +
          'class="sb-disk__row-btn sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
          'data-viewer="" ' +
          'data-viewer-type="cloud-document" ' +
          'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
          'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
          'data-viewer-extension="disk.viewer.document-item" ' +
          'data-object-id="' + escapeHtml(item.id) + '" ' +
          'data-title="' + escapeHtml(item.name) + '" ' +
          'data-actions="' + actions + '"' +
        '>Открыть</span>';
    } else {
      openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
    }

    return '' +
      '<div class="sb-disk__card ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +
           'data-id="' + escapeHtml(item.id) + '" ' +
           'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
           'data-name="' + escapeHtml(item.name) + '" ' +
           'data-download-url="' + escapeHtml(item.downloadUrl || '') + '" ' +
           'data-preview-url="' + escapeHtml(item.previewUrl || '') + '" ' +
           'data-preview-mode="' + escapeHtml(item.previewMode || '') + '">' +
          '<div class="sb-disk__card-top">' +
            '<label>' +
              '<input type="checkbox" class="sb-disk__item-check" data-id="' + escapeHtml(item.id) + '">' +
            '</label>' +
            '<span class="sb-disk__badge">' + escapeHtml(typeText) + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-name">' + escapeHtml(item.name) + '</div>' +
          '<div class="sb-disk__card-meta">' +
            '<span class="sb-disk__card-sub">Размер: ' + escapeHtml(sizeText) + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-meta">' +
            '<span class="sb-disk__card-sub">' + escapeHtml(item.updatedAt || '') + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-actions">' +
            openControl +
            (item.entityType === 'file'
              ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
              : '') +
            '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +
            '<button type="button" class="sb-disk__row-btn" data-row-action="delete">Удалить</button>' +
          '</div>' +
      '</div>';
  }).join('');
};


---

3.5. Обнови обработчик open

Найди блок:

} else if (entityType === 'file') {
  var previewUrl = row.getAttribute('data-preview-url') || '';
  var downloadUrl = row.getAttribute('data-download-url') || '';

  if (previewUrl) {
    window.open(previewUrl, '_blank');
  } else if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
}

и замени на:

} else if (entityType === 'file') {
  var previewMode = row.getAttribute('data-preview-mode') || '';
  var previewUrl = row.getAttribute('data-preview-url') || '';
  var downloadUrl = row.getAttribute('data-download-url') || '';

  if (previewMode === 'office') {
    return;
  }

  if (previewUrl) {
    window.open(previewUrl, '_blank');
  } else if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
}


---

Что получится

Теперь:

docx/xlsx/pptx будут открываться через Bitrix viewer, как в твоем рабочем примере

png/pdf/txt/html продолжат открываться через обычный browser preview

Скачать останется отдельной кнопкой

breadcrumbs уже можно держать укороченными через renderBreadcrumbs()



---

Почему у тебя было “Сообщение Задача Событие Опрос Еще”

Это было потому, что открывалась не viewer-конфигурация документа, а не та страница/роут.
С data-viewer и BX.Disk.Viewer.DocumentItem должно работать именно как в твоем примере.


---

После замены

1. сохрани DiskBitrixStorageAdapter.php


2. сохрани script.js


3. обнови страницу через Ctrl + F5


4. проверь на docx



Если после этого docx все равно не откроется как viewer, пришли HTML одной кнопки Открыть из инспектора, я сразу скажу, чего не хватает в data-атрибутах.