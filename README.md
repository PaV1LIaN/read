Да, в твоём `editor.php` **есть 2 реальные проблемы**, из-за которых поведение может быть странным/ломаться:

---

## ✅ Проблема 1: Дважды объявлен `const ec` в одном scope (это SyntaxError)

В обработчике кликов у тебя два раза:

```js
const ec = e.target.closest('[data-edit-cols2-id]');
...
const ec = e.target.closest('[data-edit-card-id]');
...
```

В JavaScript **нельзя** два раза объявлять `const` с одним именем в одном блоке `{ ... }`.
В результате весь скрипт может **вообще не выполниться** (ошибка “Identifier 'ec' has already been declared”).

### ✅ Исправление

Переименуй вторую переменную, например в `ecard`:

```js
const ecard = e.target.closest('[data-edit-card-id]');
if (ecard) { editCardBlock(parseInt(ecard.getAttribute('data-edit-card-id'), 10)); return; }
```

(а `ec` для columns2 оставь как есть)

---

## ✅ Проблема 2: У тебя нет функций для редактирования Gallery/Card через клики?

У тебя они есть (`editGalleryBlock`, `editCardBlock`) — ок.
Но **нет функции addGalleryBlock/addCardBlock в зоне видимости?** — они есть ниже, ок.

Однако, я вижу ещё одну потенциальную ошибку:

### ❗В начале ты объявляешь кнопки:

```js
const btnAddGallery = document.getElementById('btnAddGallery');
const btnAddCols2 = document.getElementById('btnAddCols2');
const btnAddSpacer = document.getElementById('btnAddSpacer');
const btnAddCard = document.getElementById('btnAddCard');
```

Это нормально.

---

# ✅ Дай я пришлю тебе готовый патч именно для твоего файла (самое надёжное)

Заменить нужно только кусок внутри:

```js
document.addEventListener('click', function (e) {
   ...
});
```

на вот этот вариант (я исправил конфликт имён и порядок):

```js
document.addEventListener('click', function (e) {
  const mvBtn = e.target.closest('[data-move-block-id]');
  if (mvBtn) {
    const id = parseInt(mvBtn.getAttribute('data-move-block-id'), 10);
    const dir = mvBtn.getAttribute('data-move-dir');
    api('block.move', { id, dir }).then(r => {
      if (!r || r.ok !== true) { BX.UI.Notification.Center.notify({ content: 'Не удалось переместить блок' }); return; }
      loadBlocks();
    }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.move' }));
    return;
  }

  const delBtn = e.target.closest('[data-del-block-id]');
  if (delBtn) {
    const id = parseInt(delBtn.getAttribute('data-del-block-id'), 10);
    BX.UI.Dialogs.MessageBox.show({
      title: 'Удалить блок #' + id + '?',
      message: 'Продолжить?',
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        api('block.delete', { id }).then(r => {
          if (!r || r.ok !== true) { BX.UI.Notification.Center.notify({ content: 'Не удалось удалить блок' }); return; }
          BX.UI.Notification.Center.notify({ content: 'Удалено' });
          mb.close();
          loadBlocks();
        }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка block.delete' }));
      }
    });
    return;
  }

  const et = e.target.closest('[data-edit-text-id]');
  if (et) { editTextBlock(parseInt(et.getAttribute('data-edit-text-id'), 10)); return; }

  const ei = e.target.closest('[data-edit-image-id]');
  if (ei) { editImageBlock(parseInt(ei.getAttribute('data-edit-image-id'), 10)); return; }

  const eb = e.target.closest('[data-edit-button-id]');
  if (eb) { editButtonBlock(parseInt(eb.getAttribute('data-edit-button-id'), 10)); return; }

  const eh = e.target.closest('[data-edit-heading-id]');
  if (eh) { editHeadingBlock(parseInt(eh.getAttribute('data-edit-heading-id'), 10)); return; }

  const ecol = e.target.closest('[data-edit-cols2-id]');
  if (ecol) { editCols2Block(parseInt(ecol.getAttribute('data-edit-cols2-id'), 10)); return; }

  const egal = e.target.closest('[data-edit-gallery-id]');
  if (egal) { editGalleryBlock(parseInt(egal.getAttribute('data-edit-gallery-id'), 10)); return; }

  const esp = e.target.closest('[data-edit-spacer-id]');
  if (esp) { editSpacerBlock(parseInt(esp.getAttribute('data-edit-spacer-id'), 10)); return; }

  const ecard = e.target.closest('[data-edit-card-id]');
  if (ecard) { editCardBlock(parseInt(ecard.getAttribute('data-edit-card-id'), 10)); return; }
});
```

---

## ✅ Ещё маленькое замечание (не критично)

Вверху у тебя:

```js
function colsGridTemplate(ratio) {
...
}
```

Она с лишними отступами, но это **не ошибка**.

---

## Что проверить после фикса

1. Открой консоль браузера — **не должно быть красной ошибки** `Identifier 'ec' has already been declared`
2. Нажми “Редактировать” у Columns2 / Gallery / Spacer / Card — должно открываться окно.

Если хочешь — я могу быстро “причесать” файл так, чтобы:

* все add/edit функции были сгруппированы,
* не было повторов,
* и было проще поддерживать дальше.
