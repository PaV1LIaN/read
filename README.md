Да, тут лучше сделать 2 точечных правки:

1. в script.js — аккуратно собрать верхнюю часть диска:

хлебные крошки слева,

Обновить и Настройки справа в той же строке;

ниже — красивая панель с поиском / сортировкой / кнопками.



2. в styles.css — нормально оформить эти новые ряды и привести кнопки Открыть к одному стилю.



И да — причина, почему у png кнопка синяя, а у doc белая, в том, что для doc/docx у тебя рендерится другой HTML-элемент (span с bitrix viewer-классами), а не обычная кнопка.
Поэтому я дам правку, чтобы у всех Открыть был единый вид.


---

1. script.js

1.1. В методе renderAll()

Найди:

DiskComponent.prototype.renderAll = function () {
  this.renderSubtitle();
  this.renderBreadcrumbs();
  this.renderItemsTable();
  this.renderItemsGrid();
  this.syncSelectedState();
};

И замени на:

DiskComponent.prototype.renderAll = function () {
  this.renderSubtitle();
  this.renderBreadcrumbs();
  this.renderItemsTable();
  this.renderItemsGrid();
  this.syncSelectedState();
  this.arrangeModernLayout();
};


---

1.2. Добавь новый метод arrangeModernLayout()

Вставь его ниже renderAll():

DiskComponent.prototype.arrangeModernLayout = function () {
  var root = this.root;

  var breadcrumbs = root.querySelector('[data-role="breadcrumbs"]');
  var refreshBtn = root.querySelector('[data-action="refresh"]');
  var settingsBtn = root.querySelector('[data-action="settings"]');

  var searchInput = root.querySelector('[data-role="search-input"]');
  var sortSelect = root.querySelector('[data-role="sort-select"]');
  var uploadBtn = root.querySelector('[data-action="upload"]');
  var createFolderBtn = root.querySelector('[data-action="create-folder"]');

  var viewButtons = Array.prototype.slice.call(root.querySelectorAll('.sb-disk__view-btn'));

  var tableContainer = root.querySelector('[data-view-container="table"]');
  var gridContainer = root.querySelector('[data-view-container="grid"]');

  var anchor =
    tableContainer ||
    gridContainer ||
    root.querySelector('[data-role="bulkbar"]') ||
    root.firstElementChild;

  if (!anchor) {
    return;
  }

  var header = root.querySelector('.sb-disk__smart-header');
  if (!header) {
    header = document.createElement('div');
    header.className = 'sb-disk__smart-header';

    var headerLeft = document.createElement('div');
    headerLeft.className = 'sb-disk__smart-header-left';

    var headerRight = document.createElement('div');
    headerRight.className = 'sb-disk__smart-header-right';

    header.appendChild(headerLeft);
    header.appendChild(headerRight);

    root.insertBefore(header, anchor);
  }

  var headerLeft = header.querySelector('.sb-disk__smart-header-left');
  var headerRight = header.querySelector('.sb-disk__smart-header-right');

  if (breadcrumbs) {
    headerLeft.appendChild(breadcrumbs);
  }

  if (refreshBtn) {
    headerRight.appendChild(refreshBtn);
  }

  if (settingsBtn) {
    headerRight.appendChild(settingsBtn);
  }

  var toolbar = root.querySelector('.sb-disk__smart-toolbar');
  if (!toolbar) {
    toolbar = document.createElement('div');
    toolbar.className = 'sb-disk__smart-toolbar';

    var toolbarLeft = document.createElement('div');
    toolbarLeft.className = 'sb-disk__smart-toolbar-left';

    var toolbarRight = document.createElement('div');
    toolbarRight.className = 'sb-disk__smart-toolbar-right';

    toolbar.appendChild(toolbarLeft);
    toolbar.appendChild(toolbarRight);

    if (header.nextSibling) {
      root.insertBefore(toolbar, header.nextSibling);
    } else {
      root.appendChild(toolbar);
    }
  }

  var toolbarLeft = toolbar.querySelector('.sb-disk__smart-toolbar-left');
  var toolbarRight = toolbar.querySelector('.sb-disk__smart-toolbar-right');

  if (searchInput) {
    toolbarLeft.appendChild(searchInput);
  }

  if (sortSelect) {
    toolbarLeft.appendChild(sortSelect);
  }

  if (uploadBtn) {
    toolbarRight.appendChild(uploadBtn);
  }

  if (createFolderBtn) {
    toolbarRight.appendChild(createFolderBtn);
  }

  viewButtons.forEach(function (btn) {
    toolbarRight.appendChild(btn);
  });
};


---

1.3. Сделай одинаковую кнопку Открыть

В renderItemsTable() найди куски, где формируется openControl.

Было:

if (item.entityType === 'folder') {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
} else if (item.previewMode === 'office') {
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
      'data-actions="' + escapeHtml(JSON.stringify([{ type: 'download' }])) + '"' +
    '>Открыть</span>';
} else {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
}

Замени на:

if (item.entityType === 'folder') {
  openControl = '<button type="button" class="sb-disk__row-btn sb-disk__row-btn--primary" data-row-action="open">Открыть</button>';
} else if (item.previewMode === 'office') {
  openControl =
    '<span ' +
      'class="sb-disk__row-btn sb-disk__row-btn--primary sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
      'data-viewer="" ' +
      'data-viewer-type="cloud-document" ' +
      'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
      'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
      'data-viewer-extension="disk.viewer.document-item" ' +
      'data-object-id="' + escapeHtml(item.id) + '" ' +
      'data-title="' + escapeHtml(item.name) + '" ' +
      'data-actions="' + escapeHtml(JSON.stringify([{ type: 'download' }])) + '"' +
    '>Открыть</span>';
} else {
  openControl = '<button type="button" class="sb-disk__row-btn sb-disk__row-btn--primary" data-row-action="open">Открыть</button>';
}


---

1.4. То же самое в renderItemsGrid()

Там тоже найди аналогичный блок openControl и замени на такой же:

if (item.entityType === 'folder') {
  openControl = '<button type="button" class="sb-disk__row-btn sb-disk__row-btn--primary" data-row-action="open">Открыть</button>';
} else if (item.previewMode === 'office') {
  openControl =
    '<span ' +
      'class="sb-disk__row-btn sb-disk__row-btn--primary sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
      'data-viewer="" ' +
      'data-viewer-type="cloud-document" ' +
      'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
      'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
      'data-viewer-extension="disk.viewer.document-item" ' +
      'data-object-id="' + escapeHtml(item.id) + '" ' +
      'data-title="' + escapeHtml(item.name) + '" ' +
      'data-actions="' + escapeHtml(JSON.stringify([{ type: 'download' }])) + '"' +
    '>Открыть</span>';
} else {
  openControl = '<button type="button" class="sb-disk__row-btn sb-disk__row-btn--primary" data-row-action="open">Открыть</button>';
}


---

2. styles.css

В конец файла добавь вот этот блок:

/* =========================================================
   DISK HEADER / TOOLBAR COMPACT
   ========================================================= */

.sb-disk__smart-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 16px;
    margin: 0 0 10px;
    padding: 0;
}

.sb-disk__smart-header-left {
    flex: 1 1 auto;
    min-width: 0;
    display: flex;
    align-items: center;
}

.sb-disk__smart-header-right {
    flex: 0 0 auto;
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
    justify-content: flex-end;
}

.sb-disk [data-role="breadcrumbs"] {
    display: flex !important;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
    margin: 0 !important;
    padding: 0 !important;
}

.sb-disk__crumb {
    display: inline-flex;
    align-items: center;
    min-height: 30px;
    padding: 0 10px;
    border: 1px solid #e3e8f2;
    border-radius: 999px;
    background: #f8fafc;
    color: #4b5563;
    font-size: 12px;
    font-weight: 700;
    cursor: pointer;
}

.sb-disk__crumb:hover {
    background: #eef4ff;
    color: #2563eb;
    border-color: #cfe0ff;
}

/* Панель поиска и кнопок */
.sb-disk__smart-toolbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px 18px;
    flex-wrap: wrap;
    margin: 0 0 14px;
    padding: 12px 14px;
    border: 1px solid #e6ebf3;
    border-radius: 16px;
    background: #fbfcfe;
}

.sb-disk__smart-toolbar-left,
.sb-disk__smart-toolbar-right {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
}

.sb-disk__smart-toolbar-left {
    flex: 1 1 360px;
    min-width: 260px;
}

.sb-disk__smart-toolbar-right {
    flex: 0 1 auto;
    justify-content: flex-end;
}

/* Красивые размеры контролов */
.sb-disk input[type="text"],
.sb-disk input[type="search"],
.sb-disk select {
    height: 36px !important;
    padding: 0 12px !important;
    border: 1px solid #dbe3ef !important;
    border-radius: 10px !important;
    background: #fff !important;
    color: #111827 !important;
    font-size: 13px !important;
}

.sb-disk [data-role="search-input"] {
    min-width: 260px;
    width: 260px;
}

.sb-disk [data-role="sort-select"] {
    min-width: 150px;
}

/* Кнопки сверху */
.sb-disk [data-action="refresh"],
.sb-disk [data-action="settings"],
.sb-disk [data-action="upload"],
.sb-disk [data-action="create-folder"],
.sb-disk .sb-disk__view-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-height: 36px;
    padding: 0 12px;
    border: 1px solid #dbe3ef;
    border-radius: 10px;
    background: #ffffff;
    color: #374151;
    font-size: 13px;
    font-weight: 700;
    cursor: pointer;
    transition: .18s ease;
}

.sb-disk [data-action="refresh"]:hover,
.sb-disk [data-action="settings"]:hover,
.sb-disk [data-action="upload"]:hover,
.sb-disk [data-action="create-folder"]:hover,
.sb-disk .sb-disk__view-btn:hover {
    border-color: #c7d7f3;
    background: #f8fbff;
    color: #1f4fd6;
}

.sb-disk .sb-disk__view-btn.is-active,
.sb-disk [data-action="upload"] {
    background: #4f6df5;
    border-color: #4f6df5;
    color: #fff;
}

.sb-disk .sb-disk__view-btn.is-active:hover,
.sb-disk [data-action="upload"]:hover {
    background: #3f5de9;
    border-color: #3f5de9;
    color: #fff;
}

/* =========================================================
   TABLE ACTION BUTTONS
   ========================================================= */

.sb-disk__actions {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 6px;
    flex-wrap: wrap;
}

.sb-disk__row-btn,
.sb-disk__viewer-btn {
    display: inline-flex !important;
    align-items: center;
    justify-content: center;
    min-height: 30px;
    padding: 0 10px;
    border: 1px solid #dbe3ef;
    border-radius: 9px;
    background: #fff;
    color: #4b5563;
    font-size: 12px;
    font-weight: 700;
    line-height: 1;
    text-decoration: none !important;
    cursor: pointer;
    transition: .18s ease;
}

.sb-disk__row-btn:hover,
.sb-disk__viewer-btn:hover {
    border-color: #c7d7f3;
    background: #f8fbff;
    color: #1f4fd6;
}

.sb-disk__row-btn--primary,
.sb-disk__viewer-btn.sb-disk__row-btn--primary {
    background: #4f6df5 !important;
    border-color: #4f6df5 !important;
    color: #fff !important;
}

.sb-disk__row-btn--primary:hover,
.sb-disk__viewer-btn.sb-disk__row-btn--primary:hover {
    background: #3f5de9 !important;
    border-color: #3f5de9 !important;
    color: #fff !important;
}

/* Таблица компактнее */
.sb-disk thead th {
    padding-top: 10px;
    padding-bottom: 10px;
    vertical-align: middle;
}

.sb-disk tbody td {
    padding-top: 10px;
    padding-bottom: 10px;
    vertical-align: middle;
}

.sb-disk__item-name {
    display: flex;
    align-items: center;
    gap: 10px;
}

.sb-disk__item-name-label {
    font-weight: 700;
    color: #1f2937;
}

/* =========================================================
   RESPONSIVE
   ========================================================= */

@media (max-width: 900px) {
    .sb-disk__smart-header,
    .sb-disk__smart-toolbar {
        align-items: flex-start;
        flex-direction: column;
    }

    .sb-disk__smart-header-right,
    .sb-disk__smart-toolbar-right {
        justify-content: flex-start;
    }

    .sb-disk [data-role="search-input"] {
        width: 100%;
        min-width: 0;
    }
}


---

Что это даст

После этих правок будет так:

1. Верхняя строка

слева: хлебные крошки

справа: Обновить / Настройки

лишняя пустота сверху уйдёт


2. Вторая строка

слева: поиск + сортировка

справа: Загрузить / Новая папка / Таблица / Плитка

всё будет смотреться ровнее и компактнее


3. Кнопка Открыть

и у png,

и у doc/docx будет одинаково синяя, потому что мы дали единый класс sb-disk__row-btn--primary



---

Если хочешь, следующим сообщением я могу уже прислать тебе готовый полный script.js целиком, чтобы ты просто вставил файл без ручного поиска кусков.