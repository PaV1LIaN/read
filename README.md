Проблема уже видна по warning:

BX.UI.Viewer.Instance.bind is not a function

То есть в твоей версии Bitrix нет такого API.
Нужно биндинг делать через BX.viewElementBind(...), а не через BX.UI.Viewer.Instance.bind(...).

Из-за этого viewer на data-viewer сейчас вообще не инициализируется, поэтому:

Открыть у docx не работает

Редактировать тоже не срабатывает нормально



---

Что исправить в script.js

1. Замени метод bindViewer()

Найди в файле:

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

И замени целиком на это:

DiskComponent.prototype.bindViewer = function () {
  if (!window.BX) {
    return;
  }

  try {
    if (typeof BX.viewElementBind === 'function') {
      BX.viewElementBind(this.root, {}, {});
      return;
    }

    console.warn('Viewer bind skipped: BX.viewElementBind is not available');
  } catch (e) {
    console.warn('Viewer bind failed', e);
  }
};


---

2. Замени метод openOfficeEditor()

Найди:

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

И замени на:

DiskComponent.prototype.openOfficeEditor = function (row) {
  if (!row || !window.BX || !BX.Disk || !BX.Disk.Viewer || !BX.Disk.Viewer.Actions) {
    return false;
  }

  var objectId = String(row.getAttribute('data-id') || '');
  var name = row.getAttribute('data-name') || '';

  if (!objectId) {
    return false;
  }

  try {
    if (typeof BX.Disk.Viewer.Actions.runActionDefaultEdit === 'function') {
      BX.Disk.Viewer.Actions.runActionDefaultEdit({
        objectId: objectId,
        name: name
      });
      return true;
    }

    console.warn('runActionDefaultEdit is not available');
    return false;
  } catch (e) {
    console.warn('Office edit failed', e);
    return false;
  }
};


---

3. Добавь fallback в обработчик edit-office

Найди блок:

var editOfficeBtn = e.target.closest('[data-row-action="edit-office"]');
if (editOfficeBtn) {
  var editRow = e.target.closest('[data-id][data-entity-type="file"]');
  if (!editRow) return;

  var previewModeForEdit = editRow.getAttribute('data-preview-mode') || '';
  if (previewModeForEdit === 'office') {
    self.openOfficeEditor(editRow);
  }
  return;
}

И замени на:

var editOfficeBtn = e.target.closest('[data-row-action="edit-office"]');
if (editOfficeBtn) {
  var editRow = e.target.closest('[data-id][data-entity-type="file"]');
  if (!editRow) return;

  var previewModeForEdit = editRow.getAttribute('data-preview-mode') || '';
  if (previewModeForEdit === 'office') {
    var ok = self.openOfficeEditor(editRow);

    if (!ok) {
      var fallbackPreviewUrl = editRow.getAttribute('data-preview-url') || '';
      if (fallbackPreviewUrl) {
        window.open(fallbackPreviewUrl, '_blank');
      }
    }
  }
  return;
}


---

Что еще проверить в консоли после правки

После Ctrl + F5 выполни:

typeof BX.viewElementBind

Должно быть:

"function"

И еще:

typeof BX.Disk !== 'undefined' && BX.Disk.Viewer && BX.Disk.Viewer.Actions ? Object.keys(BX.Disk.Viewer.Actions) : 'no disk viewer'

Это покажет, есть ли у тебя реально runActionDefaultEdit.


---

Почему сейчас не нажимается “Редактировать”

С высокой вероятностью:

viewer core был не привязан к data-viewer

а runActionDefaultEdit либо не доступен, либо не запускается без корректно инициализированного viewer-окружения


После замены на BX.viewElementBind(...) ситуация должна стать лучше.

Если после этого Открыть заработает, а Редактировать нет — пришли результат двух проверок из консоли:

typeof BX.viewElementBind

список Object.keys(BX.Disk.Viewer.Actions)