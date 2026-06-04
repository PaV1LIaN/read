Да, делаем так:

В таблице вместо колонки “Тип” будет “Добавил”.
В истории файла тоже будет видно, кем была добавлена старая версия.

Сначала меняем script.js. Если после этого в колонке будет —, значит API диска пока не отдаёт автора, тогда следующим сообщением доработаем /components/disk/api.php.


---

1. script.js — добавь helper автора

Файл:

/local/sitebuilder/components/disk/script.js

Найди блок helper-функций внизу файла, рядом с:

function getItemExtension(item) {

Перед ним вставь:

function getItemAddedByText(item) {
  if (!item) {
    return '—';
  }

  var value =
    item.createdByName ||
    item.createdByTitle ||
    item.createdByFullName ||
    item.authorName ||
    item.userName ||
    item.createdBy ||
    item.author ||
    '';

  value = String(value || '').trim();

  if (value) {
    return value;
  }

  var id =
    item.createdById ||
    item.authorId ||
    item.userId ||
    0;

  id = Number(id || 0);

  if (id > 0) {
    return 'ID ' + id;
  }

  return '—';
}


---

2. script.js — переименуй заголовок колонки

Найди метод:

DiskComponent.prototype.prepareModernUi = function () {

В конец этого метода, перед закрывающей строкой:

};

добавь:

this.renameTypeColumnToAddedBy();

Теперь перед методом:

DiskComponent.prototype.getBasePayload = function () {

вставь новый метод:

DiskComponent.prototype.renameTypeColumnToAddedBy = function () {
  var table = this.root.querySelector('table');

  if (!table) {
    return;
  }

  var headers = table.querySelectorAll('thead th');

  headers.forEach(function (th) {
    var text = String(th.textContent || '').trim().toLowerCase();

    if (text === 'тип') {
      th.textContent = 'Добавил';
    }
  });
};


---

3. script.js — замени колонку “Тип” в таблице

Найди в renderItemsTable():

var typeText = getItemTypeText(item);
var sizeText = item.entityType === 'folder' ? '—' : (item.size ? formatBytes(item.size) : '—');
var iconHtml = renderItemIcon(item);
var openControl = renderOpenControl(item);
var historyControl = self.renderHistoryControl(item);

Замени на:

var typeText = getItemTypeText(item);
var addedByText = getItemAddedByText(item);
var sizeText = item.entityType === 'folder' ? '—' : (item.size ? formatBytes(item.size) : '—');
var iconHtml = renderItemIcon(item);
var openControl = renderOpenControl(item);
var historyControl = self.renderHistoryControl(item);

Ниже найди строку:

'<td><span class="sb-disk__type-pill">' + escapeHtml(typeText) + '</span></td>' +

Замени на:

'<td><span class="sb-disk__added-by">' + escapeHtml(addedByText) + '</span></td>' +


---

4. script.js — в плитке тоже покажем автора

В renderItemsGrid() найди:

var typeText = getItemTypeText(item);
var sizeText = item.entityType === 'folder' ? 'Папка' : (item.size ? formatBytes(item.size) : '—');
var openControl = renderOpenControl(item);
var historyControl = self.renderHistoryControl(item);

Замени на:

var typeText = getItemTypeText(item);
var addedByText = getItemAddedByText(item);
var sizeText = item.entityType === 'folder' ? 'Папка' : (item.size ? formatBytes(item.size) : '—');
var openControl = renderOpenControl(item);
var historyControl = self.renderHistoryControl(item);

Найди в карточке:

'<span class="sb-disk__type-pill">' + escapeHtml(typeText) + '</span>' +

Замени на:

'<span class="sb-disk__type-pill">' + escapeHtml(typeText) + '</span>' +

Оставь так, а ниже после блока размера найди:

'<div class="sb-disk__card-meta">' +
  '<span class="sb-disk__card-sub">' + escapeHtml(sizeText) + '</span>' +
'</div>' +

И сразу после него добавь:

'<div class="sb-disk__card-meta">' +
  '<span class="sb-disk__card-sub">Добавил: ' + escapeHtml(addedByText) + '</span>' +
'</div>' +


---

5. script.js — история файла

В openHistoryModalForFile() найди внутри historyItems.map(function (item) {:

var time = self.formatHistoryDate(self.getHistoryTime(item));
var size = item.size ? formatBytes(item.size) : '—';

Замени на:

var time = self.formatHistoryDate(self.getHistoryTime(item));
var size = item.size ? formatBytes(item.size) : '—';
var addedByText = getItemAddedByText(item);

Ниже найди:

+ '      <span>' + escapeHtml(time) + '</span>'
+ '      <span>' + escapeHtml(size) + '</span>'

Замени на:

+ '      <span>' + escapeHtml(time) + '</span>'
+ '      <span>' + escapeHtml(size) + '</span>'
+ '      <span>Добавил: ' + escapeHtml(addedByText) + '</span>'


---

6. styles.css — стиль для автора

В конец файла:

/local/sitebuilder/components/disk/styles.css

добавь:

/* =========================================================
   ADDED BY COLUMN
   ========================================================= */

.sb-disk__added-by {
    display: inline-flex;
    align-items: center;
    max-width: 180px;
    min-height: 24px;
    padding: 0 9px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #374151;
    font-size: 12px;
    font-weight: 800;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}


---

7. Обнови версии подключения

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=19"></script>

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=17">

После этого сделай Ctrl + F5.

Если в колонке будет —, пришли мне файл:

/local/sitebuilder/components/disk/api.php

Тогда я добавлю туда отдачу createdByName из Bitrix Disk.