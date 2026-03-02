Да, вижу: **твой editor.php сейчас “полный” и рабочий**, и он почти совпадает по структуре с тем, что я присылал раньше. То, что мой вариант “меньше” — это просто потому, что я **убрал часть повторяющихся кусков/preview**, чтобы быстрее показать идею. Но **мы не будем заменять твой файл на “короче”** — правильнее сделать **аккуратный патч поверх твоего текущего editor.php**, чтобы ничего не потерять.

Сейчас добавим **Columns2** (две колонки) **точечно** в твой файл.

---

# ✅ ПАТЧ ДЛЯ ТВОЕГО ТЕКУЩЕГО editor.php (4 шага)

## 0) Важно

Перед этим убедись, что в `api.php` ты уже добавил тип `columns2` (я писал ранее: `block.create` и `block.update`).

---

## 1) CSS: добавь стили превью колонок

В `<style>` (после `.headingPreview ...`) вставь:

```css
/* columns2 preview */
.colsPreview { margin-top:10px; border:1px dashed #e5e7ea; border-radius:10px; padding:10px; display:grid; gap:10px; }
.colsPreview .cell { background:#fafafa; border:1px solid #eee; border-radius:10px; padding:10px; min-height:48px; }
.colsPreview pre { margin:0; background:transparent; border:none; padding:0; }
```

---

## 2) HTML: добавь кнопку “+ Columns2”

В блоке кнопок (где сейчас +Text/+Image/+Button/+Heading) добавь:

```html
<button class="ui-btn ui-btn-primary" id="btnAddCols2">+ Columns2</button>
```

---

## 3) JS: добавить переменную кнопки + функции-хелперы

В начале JS, где у тебя:

```js
const btnAddHeading = document.getElementById('btnAddHeading');
```

Добавь следом:

```js
const btnAddCols2 = document.getElementById('btnAddCols2');

function colsGridTemplate(ratio) {
  if (ratio === '33-67') return '1fr 2fr';
  if (ratio === '67-33') return '2fr 1fr';
  return '1fr 1fr';
}
```

---

## 4) renderBlocks: добавь рендер типа `columns2`

В `renderBlocks(blocks)` добавь новый блок **перед `return unknown`**:

```js
if (type === 'columns2') {
  const left = (b.content && typeof b.content.left === 'string') ? b.content.left : '';
  const right = (b.content && typeof b.content.right === 'string') ? b.content.right : '';
  const ratio = (b.content && typeof b.content.ratio === 'string') ? b.content.ratio : '50-50';
  const tpl = colsGridTemplate(ratio);

  return `
    <div class="block">
      <div class="row">
        <div><b>#${id}</b> <span class="muted">(columns2 | sort ${sort} | ratio ${BX.util.htmlspecialchars(ratio)})</span></div>
        <div class="btns">
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cols2-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        </div>
      </div>

      <div class="colsPreview" style="grid-template-columns:${tpl};">
        <div class="cell"><pre>${BX.util.htmlspecialchars(left)}</pre></div>
        <div class="cell"><pre>${BX.util.htmlspecialchars(right)}</pre></div>
      </div>
    </div>
  `;
}
```

---

## 5) Создание Columns2: добавь функцию `addCols2Block()`

Добавь **после addHeadingBlock()**:

```js
function addCols2Block() {
  BX.UI.Dialogs.MessageBox.show({
    title: 'Новый Columns2 блок',
    message: `
      <div>
        <div class="field">
          <label>Соотношение</label>
          <select id="c_ratio" class="input">
            <option value="50-50" selected>50 / 50</option>
            <option value="33-67">33 / 67</option>
            <option value="67-33">67 / 33</option>
          </select>
        </div>
        <div class="field">
          <label>Левая колонка (текст)</label>
          <textarea id="c_left" class="input" style="height:120px;"></textarea>
        </div>
        <div class="field">
          <label>Правая колонка (текст)</label>
          <textarea id="c_right" class="input" style="height:120px;"></textarea>
        </div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function (mb) {
      const ratio = (document.getElementById('c_ratio')?.value || '50-50');
      const left = (document.getElementById('c_left')?.value || '');
      const right = (document.getElementById('c_right')?.value || '');

      api('block.create', { pageId, type:'columns2', ratio, left, right })
        .then(res => {
          if (!res || res.ok !== true) {
            BX.UI.Notification.Center.notify({ content:'Не удалось создать columns2' });
            return;
          }
          BX.UI.Notification.Center.notify({ content:'Columns2 создан' });
          mb.close();
          loadBlocks();
        })
        .catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка block.create (columns2)' }));
    }
  });
}
```

---

## 6) Редактирование Columns2: функция `editCols2Block(id)`

Добавь **после editHeadingBlock()**:

```js
function editCols2Block(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

    const curRatio = blk && blk.content ? (blk.content.ratio || '50-50') : '50-50';
    const curLeft = blk && blk.content ? (blk.content.left || '') : '';
    const curRight = blk && blk.content ? (blk.content.right || '') : '';

    BX.UI.Dialogs.MessageBox.show({
      title: 'Редактировать Columns2 #' + id,
      message: `
        <div>
          <div class="field">
            <label>Соотношение</label>
            <select id="ec_ratio" class="input">
              <option value="50-50" ${curRatio==='50-50'?'selected':''}>50 / 50</option>
              <option value="33-67" ${curRatio==='33-67'?'selected':''}>33 / 67</option>
              <option value="67-33" ${curRatio==='67-33'?'selected':''}>67 / 33</option>
            </select>
          </div>
          <div class="field">
            <label>Левая колонка (текст)</label>
            <textarea id="ec_left" class="input" style="height:120px;">${BX.util.htmlspecialchars(curLeft)}</textarea>
          </div>
          <div class="field">
            <label>Правая колонка (текст)</label>
            <textarea id="ec_right" class="input" style="height:120px;">${BX.util.htmlspecialchars(curRight)}</textarea>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const ratio = (document.getElementById('ec_ratio')?.value || '50-50');
        const left = (document.getElementById('ec_left')?.value || '');
        const right = (document.getElementById('ec_right')?.value || '');

        api('block.update', { id, ratio, left, right })
          .then(r => {
            if (!r || r.ok !== true) {
              BX.UI.Notification.Center.notify({ content:'Не удалось сохранить columns2' });
              return;
            }
            BX.UI.Notification.Center.notify({ content:'Сохранено' });
            mb.close();
            loadBlocks();
          })
          .catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка block.update (columns2)' }));
      }
    });
  });
}
```

---

## 7) Подключи обработчики кликов и кнопку

### 7.1 В `document.addEventListener('click', ...)` добавь:

в конце рядом с `editHeading`:

```js
const ec = e.target.closest('[data-edit-cols2-id]');
if (ec) { editCols2Block(parseInt(ec.getAttribute('data-edit-cols2-id'), 10)); return; }
```

### 7.2 Внизу, где addEventListener на кнопки:

добавь:

```js
btnAddCols2.addEventListener('click', addCols2Block);
```

---

# Что будет дальше

После того как Columns2 появится в редакторе, следующим шагом я дам **патч для `view.php`**, чтобы колонки красиво отображались на странице (и адаптивно на мобилке — в одну колонку).

Если хочешь — скажи “давай view.php”, и я пришлю точный кусок для вставки + CSS под мобилку.
