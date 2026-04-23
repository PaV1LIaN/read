Скорее всего сейчас у тебя происходит вот что:

viewer открывается

но кнопка «Редактировать» внутри data-actions не запускает редактирование автоматически

либо document handler реально не готов к edit


То есть сам факт, что мы передали:

action: 'BX.Disk.Viewer.Actions.runActionDefaultEdit'

еще не гарантирует, что редактирование стартует.

Что проверить в первую очередь

Открой консоль на странице и выполни:

typeof BX !== 'undefined' && BX.Disk && BX.Disk.Viewer && BX.Disk.Viewer.Actions

Если вернет undefined или false, значит нужный JS для edit не загружен полностью.

Еще проверь:

typeof BX !== 'undefined' && BX.Disk && BX.Disk.Viewer && BX.Disk.Viewer.Actions && typeof BX.Disk.Viewer.Actions.runActionDefaultEdit

Если тут не function, то edit-action на странице вообще недоступен.


---

Практичный способ сделать редактирование

Вместо надежды на data-actions лучше сделать прямой вызов edit action из JS.

1. Добавь в script.js метод

DiskComponent.prototype.openOfficeEditor = function (row) {
  if (!row) return false;
  if (!window.BX || !BX.Disk || !BX.Disk.Viewer || !BX.Disk.Viewer.Actions) {
    return false;
  }

  var objectId = String(row.getAttribute('data-id') || '');
  var name = row.getAttribute('data-name') || '';

  if (!objectId) {
    return false;
  }

  try {
    BX.Disk.Viewer.Actions.runActionDefaultEdit({
      objectId: objectId,
      name: name
    });
    return true;
  } catch (e) {
    console.warn('Office edit failed', e);
    return false;
  }
};


---

2. В обработчике кликов добавь отдельную ветку для Редактировать

В bindStaticEvents() внутри:

this.root.addEventListener('click', async function (e) {

вставь до блока renameBtn вот это:

var editOfficeBtn = e.target.closest('[data-row-action="edit-office"]');
if (editOfficeBtn) {
  var editRow = e.target.closest('[data-id][data-entity-type="file"]');
  if (!editRow) return;

  var previewMode = editRow.getAttribute('data-preview-mode') || '';
  if (previewMode === 'office') {
    self.openOfficeEditor(editRow);
  }
  return;
}


---

3. В renderItemsTable() замени office-кнопку редактирования

Сейчас у тебя для office-файлов editControl сделан как span data-viewer ....
Замени его на обычную кнопку:

editControl = '<button type="button" class="sb-disk__row-btn" data-row-action="edit-office">Редактировать</button>';


---

4. В renderItemsGrid() тоже замени

editControl = '<button type="button" class="sb-disk__row-btn" data-row-action="edit-office">Редактировать</button>';


---

Почему так лучше

Потому что:

Открыть остается viewer-кнопкой через data-viewer

Редактировать запускается напрямую через BX.Disk.Viewer.Actions.runActionDefaultEdit(...)

это гораздо стабильнее, чем пытаться прокинуть edit через data-actions



---

Но есть важный момент

Если после этого редактирование все равно не стартует, то проблема уже почти точно не в компоненте, а в document handler.

У тебя раньше уже была ошибка про:

WOPI discovery

expired certificate


Если document handler Bitrix/Office Online не работает, то:

просмотр может частично жить

а редактирование не поднимется



---

Самая полезная проверка после правки

После нажатия Редактировать посмотри консоль.
Если там будет ошибка из runActionDefaultEdit, пришли ее текст.

Если хочешь, я могу сразу прислать тебе полный готовый script.js уже с отдельной рабочей кнопкой Редактировать через runActionDefaultEdit.