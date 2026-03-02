Отлично. Дальше сделаем очень нужный “строительный” блок для верстки лендингов: **Spacer / Divider** (отступ и/или линия-разделитель). Он кажется простым, но без него страницы быстро превращаются в кашу.

Сделаем тип блока: `spacer`

**content:**

```json
{ "height": 40, "line": false }
```

* `height` — высота отступа в px (10..200)
* `line` — рисовать ли тонкую линию по центру (true/false)

---

# 1) `api.php` — добавить `spacer`

## 1.1 В `block.create` добавь тип в список

В проверке `in_array($type, [...])` допиши `spacer`:

```php
if (!in_array($type, ['text','image','button','heading','columns2','gallery','spacer'], true)) { ... }
```

## 1.2 В `block.create` добавь ветку

После остальных типов:

```php
} elseif ($type === 'spacer') {
    $height = (int)($_POST['height'] ?? 40);
    if ($height < 10) $height = 10;
    if ($height > 200) $height = 200;

    $line = (string)($_POST['line'] ?? '0');
    $line = ($line === '1' || $line === 'true');

    $content = ['height' => $height, 'line' => $line];
}
```

## 1.3 В `block.update` добавь ветку

```php
} elseif ($type === 'spacer') {
    $height = (int)($_POST['height'] ?? 40);
    if ($height < 10) $height = 10;
    if ($height > 200) $height = 200;

    $line = (string)($_POST['line'] ?? '0');
    $line = ($line === '1' || $line === 'true');

    $b['content']['height'] = $height;
    $b['content']['line'] = $line;
}
```

---

# 2) `editor.php` — патч (поверх твоего текущего)

## 2.1 Добавь кнопку

В кнопки добавь:

```html
<button class="ui-btn ui-btn-primary" id="btnAddSpacer">+ Spacer</button>
```

## 2.2 В JS объяви кнопку

```js
const btnAddSpacer = document.getElementById('btnAddSpacer');
```

## 2.3 В renderBlocks добавь обработку spacer

Вставь перед unknown:

```js
if (type === 'spacer') {
  const height = (b.content && b.content.height) ? parseInt(b.content.height, 10) : 40;
  const line = (b.content && (b.content.line === true || b.content.line === 'true')) ? true : false;

  return `
    <div class="block">
      <div class="row">
        <div><b>#${id}</b> <span class="muted">(spacer | sort ${sort} | ${height}px | line ${line ? 'yes' : 'no'})</span></div>
        <div class="btns">
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-spacer-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        </div>
      </div>

      <div style="margin-top:10px; border:1px dashed #e5e7ea; border-radius:10px; padding:10px;">
        <div style="height:${height}px; position:relative; background:#fafafa; border-radius:10px;">
          ${line ? '<div style="position:absolute; left:0; right:0; top:50%; height:1px; background:#e5e7ea;"></div>' : ''}
        </div>
      </div>
    </div>
  `;
}
```

## 2.4 Функции add/edit spacer

Добавь:

```js
function addSpacerBlock() {
  BX.UI.Dialogs.MessageBox.show({
    title: 'Новый Spacer блок',
    message: `
      <div>
        <div class="field">
          <label>Высота (10..200 px)</label>
          <input id="sp_h" class="input" type="number" min="10" max="200" value="40" />
        </div>
        <div class="field">
          <label><input id="sp_line" type="checkbox" /> Рисовать линию</label>
        </div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      const height = parseInt(document.getElementById('sp_h')?.value || '40', 10);
      const line = document.getElementById('sp_line')?.checked ? '1' : '0';

      api('block.create', { pageId, type:'spacer', height, line })
        .then(res => {
          if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content:'Не удалось создать spacer' }); return; }
          BX.UI.Notification.Center.notify({ content:'Spacer создан' });
          mb.close(); loadBlocks();
        })
        .catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка block.create (spacer)' }));
    }
  });
}

function editSpacerBlock(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
    const curH = blk && blk.content ? parseInt(blk.content.height || 40, 10) : 40;
    const curLine = blk && blk.content ? (blk.content.line === true || blk.content.line === 'true') : false;

    BX.UI.Dialogs.MessageBox.show({
      title: 'Редактировать Spacer #' + id,
      message: `
        <div>
          <div class="field">
            <label>Высота (10..200 px)</label>
            <input id="esp_h" class="input" type="number" min="10" max="200" value="${curH}" />
          </div>
          <div class="field">
            <label><input id="esp_line" type="checkbox" ${curLine ? 'checked':''} /> Рисовать линию</label>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb){
        const height = parseInt(document.getElementById('esp_h')?.value || String(curH), 10);
        const line = document.getElementById('esp_line')?.checked ? '1' : '0';

        api('block.update', { id, height, line })
          .then(r => {
            if (!r || r.ok !== true) { BX.UI.Notification.Center.notify({ content:'Не удалось сохранить spacer' }); return; }
            BX.UI.Notification.Center.notify({ content:'Сохранено' });
            mb.close(); loadBlocks();
          })
          .catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка block.update (spacer)' }));
      }
    });
  });
}
```

## 2.5 Подключи клики

В общий click handler добавь:

```js
const es = e.target.closest('[data-edit-spacer-id]');
if (es) { editSpacerBlock(parseInt(es.getAttribute('data-edit-spacer-id'), 10)); return; }
```

И кнопка:

```js
btnAddSpacer.addEventListener('click', addSpacerBlock);
```

---

# 3) `view.php` — рендер spacer

## 3.1 CSS

В `<style>` добавь:

```css
.spacer { width:100%; }
.spacerLine { height:1px; background:#e5e7ea; }
```

## 3.2 В цикле блоков:

```php
<?php if ($type === 'spacer'): ?>
  <?php
    $h = (int)($b['content']['height'] ?? 40);
    if ($h < 10) $h = 10;
    if ($h > 200) $h = 200;
    $line = (bool)($b['content']['line'] ?? false);
  ?>
  <div class="spacer" style="height: <?= (int)$h ?>px; position:relative; margin-top:14px;">
    <?php if ($line): ?>
      <div class="spacerLine" style="position:absolute; left:0; right:0; top:50%;"></div>
    <?php endif; ?>
  </div>
<?php endif; ?>
```

---

## Что проверить

* Создать spacer 40px без линии → в `view.php` просто отступ
* Создать spacer 20px с линией → линия по центру
* Редактировать высоту и line → сохраняется
* На мобилке тоже норм (это просто блок)

---

Если spacer ок — дальше предложу **Card-блок (иконка/картинка + заголовок + текст + кнопка)** или **Tabs/Accordion**. Что выбираем?
