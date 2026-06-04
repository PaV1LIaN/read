Да, делаем так:

1 клик по папке — ничего не открывает
2 клика по папке — открывает папку
2 клика по файлу — открывает файл
кнопка "Открыть" — работает как раньше

1. В script.js найди этот блок и удали

Файл:

/local/sitebuilder/components/disk/script.js

Внутри bindStaticEvents() найди:

var clickableRow = e.target.closest('.sb-disk__row[data-id][data-entity-type="folder"], .sb-disk__card[data-id][data-entity-type="folder"]');
if (clickableRow && !e.target.closest('button, input, label, a, span[data-viewer]')) {
  var clickFolderId = Number(clickableRow.getAttribute('data-id') || 0);

  if (clickFolderId > 0) {
    await self.loadFolder(clickFolderId);
  }

  return;
}

Полностью удали этот кусок.


---

2. В bindStaticEvents() добавь двойной клик

Найди:

DiskComponent.prototype.bindStaticEvents = function () {
    var self = this;

Сразу после:

var self = this;

вставь:

this.root.addEventListener('dblclick', async function (e) {
  var item = e.target.closest(
    '.sb-disk__row[data-id][data-entity-type], .sb-disk__card[data-id][data-entity-type]'
  );

  if (!item || !self.root.contains(item)) {
    return;
  }

  if (e.target.closest('button, input, label, a, [data-viewer]')) {
    return;
  }

  var entityType = item.getAttribute('data-entity-type') || '';
  var entityId = Number(item.getAttribute('data-id') || 0);

  if (entityType === 'folder') {
    if (entityId > 0) {
      await self.loadFolder(entityId);
    }

    return;
  }

  if (entityType === 'file') {
    self.openFileFromElement(item);
  }
});


---

3. Проверь, что есть метод openFileFromElement

Если его ещё нет, вставь перед bindStaticEvents():

DiskComponent.prototype.openFileFromElement = function (element) {
  if (!element) {
    return;
  }

  var entityType = element.getAttribute('data-entity-type') || '';

  if (entityType !== 'file') {
    return;
  }

  var previewMode = element.getAttribute('data-preview-mode') || '';
  var previewUrl = element.getAttribute('data-preview-url') || '';
  var downloadUrl = element.getAttribute('data-download-url') || '';

  if (previewMode === 'office') {
    var viewerBtn = element.querySelector('[data-viewer]');

    if (viewerBtn) {
      viewerBtn.click();
      return;
    }
  }

  if (previewUrl) {
    window.open(previewUrl, '_blank');
    return;
  }

  if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
};


---

4. В styles.css добавь подсказки

В конец файла:

/local/sitebuilder/components/disk/styles.css

добавь:

/* =========================================================
   DOUBLE CLICK OPEN FILES AND FOLDERS
   ========================================================= */

.sb-disk__row[data-entity-type="folder"],
.sb-disk__card[data-entity-type="folder"],
.sb-disk__row[data-entity-type="file"],
.sb-disk__card[data-entity-type="file"] {
    cursor: default;
}

.sb-disk__row[data-entity-type="folder"] .sb-disk__name-cell,
.sb-disk__row[data-entity-type="file"] .sb-disk__name-cell,
.sb-disk__card[data-entity-type="folder"] .sb-disk__card-name,
.sb-disk__card[data-entity-type="file"] .sb-disk__card-name {
    cursor: pointer;
}

.sb-disk__row[data-entity-type="folder"] .sb-disk__modern-name-title::after {
    content: " · двойной клик для открытия";
    color: #9ca3af;
    font-size: 11px;
    font-weight: 600;
}

.sb-disk__row[data-entity-type="file"] .sb-disk__modern-name-title::after {
    content: " · двойной клик для открытия";
    color: #9ca3af;
    font-size: 11px;
    font-weight: 600;
}

.sb-disk__card[data-entity-type="folder"] .sb-disk__card-name::after,
.sb-disk__card[data-entity-type="file"] .sb-disk__card-name::after {
    content: "Двойной клик для открытия";
    display: block;
    margin-top: 4px;
    color: #9ca3af;
    font-size: 11px;
    font-weight: 600;
}


---

5. Обнови версии подключения

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=12"></script>

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=13">

После этого сделай Ctrl + F5.

Теперь папки и файлы будут открываться двойным кликом, а кнопка “Открыть” останется для обычного открытия одним нажатием.