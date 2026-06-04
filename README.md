Да, сделаем историю заменённых файлов без backend-переделки: при выборе “Заменить” старый файл не удаляем, а переименовываем в скрытую историю:

_history__timestamp_random__имя-файла.png

В списке такие файлы показываться не будут, но у нового файла появится кнопка “История”, где можно открыть старые версии.


---

1. script.js — добавь методы истории

Файл:

/local/sitebuilder/components/disk/script.js

Найди место перед:

DiskComponent.prototype.bindStaticEvents = function () {

и вставь:

DiskComponent.prototype.isHistoryItem = function (item) {
  return String(item && item.name ? item.name : '').indexOf('_history__') === 0;
};

DiskComponent.prototype.getHistoryOriginalName = function (item) {
  var name = String(item && item.name ? item.name : '');

  if (name.indexOf('_history__') !== 0) {
    return name;
  }

  var rest = name.slice('_history__'.length);
  var sepIndex = rest.indexOf('__');

  if (sepIndex < 0) {
    return name;
  }

  return rest.slice(sepIndex + 2);
};

DiskComponent.prototype.getHistoryTime = function (item) {
  var name = String(item && item.name ? item.name : '');

  if (name.indexOf('_history__') !== 0) {
    return 0;
  }

  var rest = name.slice('_history__'.length);
  var sepIndex = rest.indexOf('__');

  if (sepIndex < 0) {
    return 0;
  }

  var timePart = rest.slice(0, sepIndex);
  var timestamp = Number(String(timePart).split('_')[0] || 0);

  return timestamp > 0 ? timestamp : 0;
};

DiskComponent.prototype.formatHistoryDate = function (timestamp) {
  timestamp = Number(timestamp || 0);

  if (!timestamp) {
    return 'Старая версия';
  }

  var date = new Date(timestamp);

  return date.toLocaleString('ru-RU');
};

DiskComponent.prototype.buildHistoryFileName = function (originalName) {
  originalName = String(originalName || '').trim();

  if (!originalName) {
    originalName = 'file';
  }

  var stamp = Date.now() + '_' + Math.floor(Math.random() * 100000);

  return '_history__' + stamp + '__' + originalName;
};

DiskComponent.prototype.getHistoryItemsForFile = function (fileName) {
  fileName = String(fileName || '').trim().toLowerCase();

  if (!fileName) {
    return [];
  }

  return this.state.items
    .filter(function (item) {
      if (!this.isHistoryItem(item)) {
        return false;
      }

      return String(this.getHistoryOriginalName(item) || '').trim().toLowerCase() === fileName;
    }, this)
    .sort(function (a, b) {
      return this.getHistoryTime(b) - this.getHistoryTime(a);
    }.bind(this));
};

DiskComponent.prototype.renameDiskItem = async function (item, newName) {
  var payload = this.getBasePayload();

  payload.entityType = item.entityType || 'file';
  payload.entityId = Number(item.id || 0);
  payload.newName = newName;
  payload.sessid = this.getSessid();

  var res = await this.api('rename', payload);

  if (!res || !res.ok) {
    throw new Error((res && (res.message || res.error)) || 'RENAME_ERROR');
  }

  return res;
};

DiskComponent.prototype.archiveExistingFileToHistory = async function (existingItem) {
  if (!existingItem || String(existingItem.entityType || '') !== 'file') {
    return;
  }

  var oldName = String(existingItem.name || '').trim();

  if (!oldName) {
    return;
  }

  var historyName = this.buildHistoryFileName(oldName);

  await this.renameDiskItem(existingItem, historyName);

  existingItem.name = historyName;
};

DiskComponent.prototype.renderHistoryControl = function (item) {
  if (!item || String(item.entityType || '') !== 'file') {
    return '';
  }

  if (this.isHistoryItem(item)) {
    return '';
  }

  var historyItems = this.getHistoryItemsForFile(item.name);

  if (!historyItems.length) {
    return '';
  }

  return '<button type="button" class="sb-disk__row-btn" data-row-action="history">История</button>';
};

DiskComponent.prototype.openHistoryModalForFile = function (fileName) {
  var self = this;
  var historyItems = this.getHistoryItemsForFile(fileName);

  if (!historyItems.length) {
    alert('Истории замен для этого файла пока нет');
    return;
  }

  var modal = document.createElement('div');
  modal.className = 'sb-disk-history-modal';

  modal.innerHTML = ''
    + '<div class="sb-disk-history-modal__backdrop" data-history-action="close"></div>'
    + '<div class="sb-disk-history-modal__dialog">'
    + '  <div class="sb-disk-history-modal__head">'
    + '    <div>'
    + '      <div class="sb-disk-history-modal__title">История файла</div>'
    + '      <div class="sb-disk-history-modal__subtitle">' + escapeHtml(fileName) + '</div>'
    + '    </div>'
    + '    <button type="button" class="sb-disk-history-modal__close" data-history-action="close">×</button>'
    + '  </div>'
    + '  <div class="sb-disk-history-modal__body">'
    + historyItems.map(function (item) {
        var time = self.formatHistoryDate(self.getHistoryTime(item));
        var size = item.size ? formatBytes(item.size) : '—';

        return ''
          + '<div class="sb-disk-history-item" data-history-id="' + escapeHtml(item.id) + '">'
          + '  <div class="sb-disk-history-item__main">'
          + '    <div class="sb-disk-history-item__name">' + escapeHtml(self.getHistoryOriginalName(item)) + '</div>'
          + '    <div class="sb-disk-history-item__meta">'
          + '      <span>' + escapeHtml(time) + '</span>'
          + '      <span>' + escapeHtml(size) + '</span>'
          + '    </div>'
          + '  </div>'
          + '  <div class="sb-disk-history-item__actions">'
          + '    <button type="button" class="sb-disk-history-btn is-primary" data-history-action="open" data-history-id="' + escapeHtml(item.id) + '">Открыть</button>'
          + (item.downloadUrl
              ? '    <button type="button" class="sb-disk-history-btn" data-history-action="download" data-history-id="' + escapeHtml(item.id) + '">Скачать</button>'
              : '')
          + '  </div>'
          + '</div>';
      }).join('')
    + '  </div>'
    + '</div>';

  function findHistoryItem(id) {
    id = Number(id || 0);

    for (var i = 0; i < historyItems.length; i++) {
      if (Number(historyItems[i].id || 0) === id) {
        return historyItems[i];
      }
    }

    return null;
  }

  function close() {
    if (modal && modal.parentNode) {
      modal.parentNode.removeChild(modal);
    }

    document.removeEventListener('keydown', onKeyDown);
  }

  function onKeyDown(e) {
    if (e.key === 'Escape') {
      close();
    }
  }

  modal.addEventListener('click', function (e) {
    var btn = e.target.closest('[data-history-action]');

    if (!btn) {
      return;
    }

    var action = btn.getAttribute('data-history-action');

    if (action === 'close') {
      close();
      return;
    }

    var id = Number(btn.getAttribute('data-history-id') || 0);
    var item = findHistoryItem(id);

    if (!item) {
      return;
    }

    if (action === 'open') {
      if (item.previewUrl) {
        window.open(item.previewUrl, '_blank');
        return;
      }

      if (item.downloadUrl) {
        window.open(item.downloadUrl, '_blank');
      }

      return;
    }

    if (action === 'download') {
      if (item.downloadUrl) {
        window.open(item.downloadUrl, '_blank');
      }
    }
  });

  document.addEventListener('keydown', onKeyDown);
  document.body.appendChild(modal);
};


---

2. script.js — замени findExistingFileByName

Найди функцию:

DiskComponent.prototype.findExistingFileByName = function (fileName) {

и замени её целиком:

DiskComponent.prototype.findExistingFileByName = function (fileName) {
  fileName = String(fileName || '').trim().toLowerCase();

  if (!fileName) {
    return null;
  }

  for (var i = 0; i < this.state.items.length; i++) {
    var item = this.state.items[i];

    if (this.isHistoryItem(item)) {
      continue;
    }

    if (String(item.entityType || '').toLowerCase() !== 'file') {
      continue;
    }

    if (String(item.name || '').trim().toLowerCase() === fileName) {
      return item;
    }
  }

  return null;
};


---

3. script.js — замени getDisplayItems

Найди:

DiskComponent.prototype.getDisplayItems = function () {

и замени целиком:

DiskComponent.prototype.getDisplayItems = function () {
  var folders = [];
  var files = [];

  this.state.items.forEach(function (item) {
    if (this.isHistoryItem(item)) {
      return;
    }

    if (String(item.entityType || '').toLowerCase() === 'folder') {
      folders.push(item);
    } else {
      files.push(item);
    }
  }, this);

  return folders.concat(files);
};


---

4. script.js — замени логику “Заменить”

В обработчике загрузки найди кусок:

if (decision.action === 'replace') {
  await self.deleteDiskItems([{
    id: Number(existingItem.id || 0),
    entityType: 'file'
  }]);

  self.state.items = self.state.items.filter(function (item) {
    return Number(item.id || 0) !== Number(existingItem.id || 0);
  });

  preparedFiles.push(file);
  continue;
}

Замени на:

if (decision.action === 'replace') {
  await self.archiveExistingFileToHistory(existingItem);

  preparedFiles.push(file);
  continue;
}

Теперь старый файл не удаляется, а уходит в историю.


---

5. script.js — добавь кнопку “История” в таблицу

В renderItemsTable() перед:

var openControl = renderOpenControl(item);

добавь:

var historyControl = self.renderHistoryControl(item);

Если в начале renderItemsTable() ещё нет:

var self = this;

добавь перед tbody.innerHTML:

var self = this;

В действиях найди:

+ (item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +

Замени на:

+ (item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
historyControl +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +


---

6. script.js — добавь кнопку “История” в плитку

В renderItemsGrid() перед:

var openControl = renderOpenControl(item);

добавь:

var historyControl = self.renderHistoryControl(item);

Если в начале renderItemsGrid() ещё нет:

var self = this;

добавь перед container.innerHTML:

var self = this;

В действиях найди:

+ (item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +

Замени на:

+ (item.entityType === 'file'
  ? '<button type="button" class="sb-disk__row-btn" data-row-action="download">Скачать</button>'
  : '') +
historyControl +
'<button type="button" class="sb-disk__row-btn" data-row-action="rename">Переим.</button>' +


---

7. script.js — обработчик кнопки “История”

Внутри общего click-обработчика найди блок:

var downloadBtn = e.target.closest('[data-row-action="download"]');

Перед ним вставь:

var historyBtn = e.target.closest('[data-row-action="history"]');
if (historyBtn) {
  var historyRow = e.target.closest('[data-id][data-entity-type="file"]');

  if (!historyRow) {
    return;
  }

  var fileName = historyRow.getAttribute('data-name') || '';

  self.openHistoryModalForFile(fileName);
  return;
}


---

8. styles.css — стили истории

В конец:

/local/sitebuilder/components/disk/styles.css

добавь:

/* =========================================================
   FILE HISTORY MODAL
   ========================================================= */

.sb-disk-history-modal {
    position: fixed;
    inset: 0;
    z-index: 21000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
}

.sb-disk-history-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(15, 23, 42, .48);
    backdrop-filter: blur(5px);
}

.sb-disk-history-modal__dialog {
    position: relative;
    width: min(640px, 100%);
    max-height: calc(100vh - 48px);
    overflow: hidden;
    border: 1px solid #e5e7eb;
    border-radius: 22px;
    background: #fff;
    box-shadow: 0 28px 80px rgba(15, 23, 42, .30);
}

.sb-disk-history-modal__head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding: 20px 22px;
    border-bottom: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-disk-history-modal__title {
    color: #111827;
    font-size: 20px;
    font-weight: 900;
    line-height: 1.25;
}

.sb-disk-history-modal__subtitle {
    margin-top: 5px;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.45;
    word-break: break-word;
}

.sb-disk-history-modal__close {
    width: 34px;
    height: 34px;
    border: 1px solid #e5e7eb;
    border-radius: 11px;
    background: #fff;
    color: #6b7280;
    cursor: pointer;
    font-size: 22px;
    line-height: 1;
}

.sb-disk-history-modal__close:hover {
    background: #f3f4f6;
    color: #111827;
}

.sb-disk-history-modal__body {
    max-height: calc(100vh - 170px);
    overflow: auto;
    padding: 16px;
}

.sb-disk-history-item {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 12px;
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    background: #fff;
}

.sb-disk-history-item + .sb-disk-history-item {
    margin-top: 10px;
}

.sb-disk-history-item:hover {
    border-color: #c7d2fe;
    background: #f8fbff;
}

.sb-disk-history-item__main {
    min-width: 0;
}

.sb-disk-history-item__name {
    color: #111827;
    font-size: 14px;
    font-weight: 900;
    word-break: break-word;
}

.sb-disk-history-item__meta {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 6px;
}

.sb-disk-history-item__meta span {
    display: inline-flex;
    min-height: 22px;
    align-items: center;
    padding: 0 8px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #6b7280;
    font-size: 12px;
    font-weight: 700;
}

.sb-disk-history-item__actions {
    display: flex;
    align-items: center;
    gap: 8px;
    flex: 0 0 auto;
}

.sb-disk-history-btn {
    min-height: 34px;
    padding: 0 12px;
    border: 1px solid #dbe3ef;
    border-radius: 11px;
    background: #fff;
    color: #374151;
    font-size: 13px;
    font-weight: 800;
    cursor: pointer;
}

.sb-disk-history-btn:hover {
    border-color: #c7d2fe;
    background: #eef2ff;
    color: var(--disk-accent, #2563eb);
}

.sb-disk-history-btn.is-primary {
    border-color: var(--disk-accent, #2563eb);
    background: var(--disk-accent, #2563eb);
    color: #fff;
}

.sb-disk-history-btn.is-primary:hover {
    background: var(--disk-accent-hover, #3f5de9);
    border-color: var(--disk-accent-hover, #3f5de9);
    color: #fff;
}

@media (max-width: 700px) {
    .sb-disk-history-modal {
        align-items: flex-start;
        padding: 12px;
    }

    .sb-disk-history-item {
        align-items: stretch;
        flex-direction: column;
    }

    .sb-disk-history-item__actions {
        justify-content: flex-start;
        flex-wrap: wrap;
    }
}


---

9. Обнови версии подключения

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=17"></script>

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=16">

После Ctrl + F5 схема будет такая:

Загружаешь файл с тем же именем
→ выбираешь “Заменить”
→ старый файл скрывается в историю
→ новый файл загружается
→ у нового файла появляется кнопка “История”
→ в истории можно открыть или скачать старую версию