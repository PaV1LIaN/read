Значит BX.Disk.Viewer у тебя не открывается через Actions.show(...), и код уходит в fallback на previewUrl, а он у тебя ведет на /disk/downloadFile/..., поэтому и скачивает.

Раз в твоем рабочем примере просмотр запускается через data-viewer, надо делать точно так же и в компоненте.

Что исправить

1. В script.js больше не открываем office-файл через openOfficeViewer()

Вместо этого будем:

рендерить для office-файла скрытый viewer-элемент с теми же data-*, что в твоем рабочем коде

по кнопке Открыть программно кликать по нему



---

2. Замени renderItemsTable() целиком

Файл:

/local/sitebuilder/components/disk/script.js

Замени метод renderItemsTable на этот:

DiskComponent.prototype.renderItemsTable = function () {
  var tbody = this.root.querySelector('[data-role="items-table"]');
  if (!tbody) return;

  tbody.innerHTML = this.state.items.map(function (item) {
    var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
    var sizeText = item.size ? formatBytes(item.size) : '';
    var badge = item.entityType === 'folder'
      ? '<span class="sb-disk__badge">Папка</span>'
      : '<span class="sb-disk__badge">' + escapeHtml(item.extension || 'Файл') + '</span>';

    var officeViewerHtml = '';

    if (item.entityType === 'file' && item.previewMode === 'office') {
      var actions = escapeHtml(JSON.stringify([{ type: 'download' }]));

      officeViewerHtml =
        '<span ' +
          'class="sb-disk__office-viewer-trigger" ' +
          'style="display:none" ' +
          'data-viewer="" ' +
          'data-viewer-type="cloud-document" ' +
          'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
          'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
          'data-viewer-extension="disk.viewer.document-item" ' +
          'data-object-id="' + escapeHtml(item.id) + '" ' +
          'data-title="' + escapeHtml(item.name) + '" ' +
          'data-actions="' + actions + '"' +
        '></span>';
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
              officeViewerHtml +
            '</div>' +
          '</td>' +
          '<td>' + escapeHtml(typeText) + '</td>' +
          '<td>' + escapeHtml(sizeText) + '</td>' +
          '<td>' + escapeHtml(item.updatedAt || '') + '</td>' +
          '<td>' +
            '<div class="sb-disk__actions">' +
              '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>' +
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

3. Замени renderItemsGrid() целиком

DiskComponent.prototype.renderItemsGrid = function () {
  var container = this.root.querySelector('[data-view-container="grid"]');
  if (!container) return;

  container.classList.add('sb-disk__grid');

  container.innerHTML = this.state.items.map(function (item) {
    var typeText = item.entityType === 'folder' ? 'Папка' : (item.extension || 'Файл');
    var sizeText = item.size ? formatBytes(item.size) : '—';

    var officeViewerHtml = '';

    if (item.entityType === 'file' && item.previewMode === 'office') {
      var actions = escapeHtml(JSON.stringify([{ type: 'download' }]));

      officeViewerHtml =
        '<span ' +
          'class="sb-disk__office-viewer-trigger" ' +
          'style="display:none" ' +
          'data-viewer="" ' +
          'data-viewer-type="cloud-document" ' +
          'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
          'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
          'data-viewer-extension="disk.viewer.document-item" ' +
          'data-object-id="' + escapeHtml(item.id) + '" ' +
          'data-title="' + escapeHtml(item.name) + '" ' +
          'data-actions="' + actions + '"' +
        '></span>';
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
          officeViewerHtml +
          '<div class="sb-disk__card-meta">' +
            '<span class="sb-disk__card-sub">Размер: ' + escapeHtml(sizeText) + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-meta">' +
            '<span class="sb-disk__card-sub">' + escapeHtml(item.updatedAt || '') + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-actions">' +
            '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>' +
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

4. В обработчике open замени блок для файлов

Найди кусок:

if (previewMode === 'office') {
  if (!self.openOfficeViewer(row) && previewUrl) {
    window.open(previewUrl, '_blank');
  }
  return;
}

И замени на:

if (previewMode === 'office') {
  var hiddenViewer = row.querySelector('.sb-disk__office-viewer-trigger');
  if (hiddenViewer) {
    hiddenViewer.click();
    return;
  }

  if (previewUrl) {
    window.open(previewUrl, '_blank');
  }
  return;
}


---

5. Метод openOfficeViewer() больше не нужен

Можешь удалить его целиком из файла, чтобы не путался.


---

Почему это сработает

Потому что ты уже показал рабочий способ в другом коде:

элемент со span

data-viewer

data-viewer-type="cloud-document"

data-viewer-type-class="BX.Disk.Viewer.DocumentItem"

data-viewer-extension="disk.viewer.document-item"


То есть нужно не пытаться открыть viewer вручную через JS API, а воспроизвести тот же HTML-паттерн и кликнуть по нему.


---

После этого

1. сохрани script.js


2. сделай Ctrl + F5


3. снова нажми Открыть у docx



Если все равно не откроется, пришли HTML строки этого docx из инспектора, и я скажу, какого data-* еще не хватает.