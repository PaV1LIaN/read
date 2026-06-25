Да, в твоём script.js ошибка вот тут:

openControl +
item.entityType === 'file'

Нужно обернуть условие в скобки. Сейчас JS ломает сборку строки.

Замени целиком только функцию renderItemsTable на эту:

DiskComponent.prototype.renderItemsTable = function () {
  var tbody = this.root.querySelector('[data-role="items-table"]');
  if (!tbody) {
    return;
  }

  var self = this;

  tbody.innerHTML = this.getDisplayItems().map(function (item) {
    var typeText = getItemTypeText(item);
    var addedByText = getItemAddedByText(item);
    var sizeText = item.entityType === 'folder' ? '—' : (item.size ? formatBytes(item.size) : '—');
    var iconHtml = renderItemIcon(item);
    var openControl = renderOpenControl(item);
    var historyControl = self.renderHistoryControl(item);

    return '' +
      '<tr class="sb-disk__row ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +
        'data-id="' + escapeHtml(item.id) + '" ' +
        'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
        'data-name="' + escapeHtml(item.name) + '" ' +
        'data-download-url="' + escapeHtml(item.downloadUrl || '') + '" ' +
        'data-preview-url="' + escapeHtml(item.previewUrl || '') + '" ' +
        'data-preview-mode="' + escapeHtml(item.previewMode || '') + '">' +
          '<td class="sb-disk__check-cell">' +
            '<input type="checkbox" class="sb-disk__item-check" data-id="' + escapeHtml(item.id) + '">' +
          '</td>' +
          '<td class="sb-disk__name-cell">' +
            '<div class="sb-disk__modern-name">' +
              iconHtml +
              '<div class="sb-disk__modern-name-main">' +
                '<div class="sb-disk__modern-name-title">' + escapeHtml(item.name) + '</div>' +
                '<div class="sb-disk__modern-name-sub">' + escapeHtml(typeText) + '</div>' +
              '</div>' +
            '</div>' +
          '</td>' +
          '<td><span class="sb-disk__added-by">' + escapeHtml(addedByText) + '</span></td>' +
          '<td>' + escapeHtml(sizeText) + '</td>' +
          '<td>' + escapeHtml(item.updatedAt || '—') + '</td>' +
          '<td>' +
            '<div class="sb-disk__actions">' +
              openControl +
              historyControl +
              (item.entityType === 'file'
                ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
                : '') +
              (isArchiveItem(item)
                ? '<button type="button" class="sb-disk__row-btn" data-row-action="unpack">Распаковать</button>'
                : '') +
              '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +
              '<button type="button" class="sb-disk__row-btn is-danger" data-row-action="delete">Удалить</button>' +
            '</div>' +
          '</td>' +
      '</tr>';
  }).join('');
};

И ещё лучше сразу замени целиком функцию renderItemsGrid, чтобы там тоже была кнопка История и Распаковать нормально:

DiskComponent.prototype.renderItemsGrid = function () {
  var container = this.root.querySelector('[data-view-container="grid"]');
  if (!container) {
    return;
  }

  container.classList.add('sb-disk__grid');

  var self = this;

  container.innerHTML = this.getDisplayItems().map(function (item) {
    var typeText = getItemTypeText(item);
    var addedByText = getItemAddedByText(item);
    var sizeText = item.entityType === 'folder' ? 'Папка' : (item.size ? formatBytes(item.size) : '—');
    var openControl = renderOpenControl(item);
    var historyControl = self.renderHistoryControl(item);

    return '' +
      '<div class="sb-disk__card ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +
           'data-id="' + escapeHtml(item.id) + '" ' +
           'data-entity-type="' + escapeHtml(item.entityType) + '" ' +
           'data-name="' + escapeHtml(item.name) + '" ' +
           'data-download-url="' + escapeHtml(item.downloadUrl || '') + '" ' +
           'data-preview-url="' + escapeHtml(item.previewUrl || '') + '" ' +
           'data-preview-mode="' + escapeHtml(item.previewMode || '') + '">' +
          '<div class="sb-disk__card-top">' +
            '<label class="sb-disk__card-check">' +
              '<input type="checkbox" class="sb-disk__item-check" data-id="' + escapeHtml(item.id) + '">' +
            '</label>' +
            '<span class="sb-disk__type-pill">' + escapeHtml(typeText) + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-preview">' +
            renderItemIcon(item) +
          '</div>' +
          '<div class="sb-disk__card-name">' + escapeHtml(item.name) + '</div>' +
          '<div class="sb-disk__card-meta">' +
            '<span class="sb-disk__card-sub">' + escapeHtml(sizeText) + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-meta">' +
            '<span class="sb-disk__card-sub">Добавил: ' + escapeHtml(addedByText) + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-meta">' +
            '<span class="sb-disk__card-sub">' + escapeHtml(item.updatedAt || '') + '</span>' +
          '</div>' +
          '<div class="sb-disk__card-actions">' +
            openControl +
            historyControl +
            (item.entityType === 'file'
              ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
              : '') +
            (isArchiveItem(item)
              ? '<button type="button" class="sb-disk__row-btn" data-row-action="unpack">Распаковать</button>'
              : '') +
            '<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +
            '<button type="button" class="sb-disk__row-btn is-danger" data-row-action="delete">Удалить</button>' +
          '</div>' +
      '</div>';
  }).join('');
};

После замены сделай Ctrl + F5.

Если страница перестала открываться из-за JS, открой консоль. Сейчас эта правка должна убрать синтаксическую ошибку.