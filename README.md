Да. Для твоего disk это лучше всего делать через штатный Bitrix viewer/action edit, как в твоем рабочем примере.

То есть логика должна быть такой:

для doc/docx/xls/xlsx/ppt/pptx/...

Открыть → просмотр

Редактировать → BX.Disk.Viewer.Actions.runActionDefaultEdit


для обычных файлов

Открыть → browser preview

Редактировать не показывать



Ниже даю, что именно менять.


---

1. Обнови DiskBitrixStorageAdapter.php

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
            ? $this->buildViewerSrc($file)
            : $this->buildPreviewUrl($context, $file),
        'previewMode' => $isOffice ? 'office' : 'browser',
        'canEdit' => $isOffice,
        'createdAt' => $this->normalizeDate($file->getCreateTime()),
        'updatedAt' => $this->normalizeDate($file->getUpdateTime()),
        'createdBy' => (int)$file->getCreatedBy(),
    ];
}


---

2. Обнови script.js

Файл:

/local/sitebuilder/components/disk/script.js

Замени renderItemsTable() целиком на это:

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
    var editControl = '';

    if (item.entityType === 'folder') {
      openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
    } else if (item.previewMode === 'office') {
      var openActions = escapeHtml(JSON.stringify([{ type: 'download' }]));
      var editActions = escapeHtml(JSON.stringify([
        { type: 'download' },
        {
          type: 'edit',
          action: 'BX.Disk.Viewer.Actions.runActionDefaultEdit',
          params: {
            objectId: String(item.id),
            name: item.name
          }
        }
      ]));

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
          'data-actions="' + openActions + '"' +
        '>Открыть</span>';

      editControl =
        '<span ' +
          'class="sb-disk__row-btn sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
          'data-viewer="" ' +
          'data-viewer-type="cloud-document" ' +
          'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
          'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
          'data-viewer-extension="disk.viewer.document-item" ' +
          'data-object-id="' + escapeHtml(item.id) + '" ' +
          'data-title="' + escapeHtml(item.name) + '" ' +
          'data-actions="' + editActions + '"' +
        '>Редактировать</span>';
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
              editControl +
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

Замени renderItemsGrid() целиком на это:

DiskComponent.prototype.renderItemsGrid = function () {
  var container = this.root.querySelector('[data-view-container="grid"]');
  if (!container) return;

  container.classList.add('sb-disk__grid');

  container.innerHTML = this.state.items.map(function (item) {
    var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
    var sizeText = item.size ? formatBytes(item.size) : '—';

    var openControl = '';
    var editControl = '';

    if (item.entityType === 'folder') {
      openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
    } else if (item.previewMode === 'office') {
      var openActions = escapeHtml(JSON.stringify([{ type: 'download' }]));
      var editActions = escapeHtml(JSON.stringify([
        { type: 'download' },
        {
          type: 'edit',
          action: 'BX.Disk.Viewer.Actions.runActionDefaultEdit',
          params: {
            objectId: String(item.id),
            name: item.name
          }
        }
      ]));

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
          'data-actions="' + openActions + '"' +
        '>Открыть</span>';

      editControl =
        '<span ' +
          'class="sb-disk__row-btn sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
          'data-viewer="" ' +
          'data-viewer-type="cloud-document" ' +
          'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
          'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
          'data-viewer-extension="disk.viewer.document-item" ' +
          'data-object-id="' + escapeHtml(item.id) + '" ' +
          'data-title="' + escapeHtml(item.name) + '" ' +
          'data-actions="' + editActions + '"' +
        '>Редактировать</span>';
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
            editControl +
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

3. Что важно

Чтобы редактирование работало, у вас должен быть реально настроен document handler:

Office Online / Microsoft Office Online / Bitrix24 Docs

и он должен быть живой


Если раньше у тебя уже была ошибка с WOPI discovery и просроченным сертификатом, то редактирование не заработает, пока не починен document handler.

То есть UI-кнопку мы сейчас добавим правильно, но если инфраструктура handler-а сломана, Bitrix viewer может не открыть редактор.


---

4. Права

Если хочешь, чтобы Редактировать показывалось не всем, а только тем, у кого есть право редактирования, тогда следующим шагом можно сделать это через:

permissions.canRename или

отдельный permissions.canEditFile


Сейчас я оставил показ для всех office-файлов.


---

5. После замены

1. сохрани оба файла


2. сделай Ctrl + F5


3. проверь docx



Если после этого кнопка Редактировать покажется, но редактор не откроется, тогда проблема уже не в компоненте, а в document handler.