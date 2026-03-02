Отлично. Дальше самый полезный “тильда-шаг”: **Header-блок (заголовок)** + **разделитель секций**. Это даёт структуру странице (как в реальных лендингах).

Сделаем новый тип блока: `heading`
Поля:

* `text` — текст заголовка
* `level` — `h1|h2|h3` (по умолчанию h2)
* `align` — `left|center|right` (по умолчанию left)

Ниже — конкретные изменения (как делали с button): **только в двух местах api.php + editor.php + view.php**.

---

## 1) `api.php` — добавить `heading`

### 1.1 `block.create`: разрешить тип

В `block.create` (где `in_array($type, ...)`) добавь `heading`:

```php
if (!in_array($type, ['text','image','button','heading'], true)) { ... }
```

### 1.2 `block.create`: добавить ветку `heading`

В твоём `block.create` после `button` добавь:

```php
} elseif ($type === 'heading') {
    $text = trim((string)($_POST['text'] ?? ''));
    $level = strtolower(trim((string)($_POST['level'] ?? 'h2')));
    $align = strtolower(trim((string)($_POST['align'] ?? 'left')));

    if ($text === '') {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
    if (!in_array($align, ['left','center','right'], true)) $align = 'left';

    $content = ['text' => $text, 'level' => $level, 'align' => $align];
}
```

### 1.3 `block.update`: добавить ветку `heading`

В `block.update` добавь:

```php
} elseif ($type === 'heading') {
    $text = trim((string)($_POST['text'] ?? ''));
    $level = strtolower(trim((string)($_POST['level'] ?? 'h2')));
    $align = strtolower(trim((string)($_POST['align'] ?? 'left')));

    if ($text === '') {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
    if (!in_array($align, ['left','center','right'], true)) $align = 'left';

    $b['content']['text'] = $text;
    $b['content']['level'] = $level;
    $b['content']['align'] = $align;
}
```

---

## 2) `editor.php` — добавить кнопку “+ Heading” и редактирование

Если хочешь, я пришлю **готовый editor.php целиком**, но проще так:

* Добавь кнопку рядом с другими:

```html
<button class="ui-btn ui-btn-primary" id="btnAddHeading">+ Heading</button>
```

* В JS:

  * `const btnAddHeading = document.getElementById('btnAddHeading');`
  * `btnAddHeading.addEventListener('click', addHeadingBlock);`

* Реализация `addHeadingBlock()` (готовая):

```js
function addHeadingBlock() {
  BX.UI.Dialogs.MessageBox.show({
    title: 'Новый Heading блок',
    message: `
      <div>
        <div class="field">
          <label>Текст</label>
          <input id="h_text" class="input" placeholder="например: О нас" />
        </div>
        <div class="field">
          <label>Уровень</label>
          <select id="h_level" class="input">
            <option value="h1">h1</option>
            <option value="h2" selected>h2</option>
            <option value="h3">h3</option>
          </select>
        </div>
        <div class="field">
          <label>Выравнивание</label>
          <select id="h_align" class="input">
            <option value="left" selected>left</option>
            <option value="center">center</option>
            <option value="right">right</option>
          </select>
        </div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function (mb) {
      const text = (document.getElementById('h_text')?.value || '').trim();
      const level = (document.getElementById('h_level')?.value || 'h2');
      const align = (document.getElementById('h_align')?.value || 'left');

      if (!text) { BX.UI.Notification.Center.notify({ content:'Введите текст' }); return; }

      api('block.create', { pageId, type:'heading', text, level, align }).then(res => {
        if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content:'Не удалось создать heading' }); return; }
        BX.UI.Notification.Center.notify({ content:'Heading создан' });
        mb.close();
        loadBlocks();
      });
    }
  });
}
```

* В `renderBlocks()` добавь отображение `heading` и кнопку редактировать:

  * показываем `text`, `level`, `align`
  * кнопка `data-edit-heading-id="..."`

* И функция `editHeadingBlock(id)` аналогично button.

Если хочешь без ручных вставок — скажи, и я **пришлю editor.php готовым файлом** (как делали раньше).

---

## 3) `view.php` — рендер heading

В цикле блоков добавь:

```php
<?php if ($type === 'heading'): ?>
  <?php
    $text = (string)($b['content']['text'] ?? '');
    $level = (string)($b['content']['level'] ?? 'h2');
    $align = (string)($b['content']['align'] ?? 'left');
    if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
    if (!in_array($align, ['left','center','right'], true)) $align = 'left';
  ?>
  <<?=h($level)?> style="margin-top:16px; text-align:<?=h($align)?>;">
    <?=h($text)?>
  </<?=h($level)?>>
<?php endif; ?>
```

---

## Что выбрать: “готовый editor.php” или “вставками”?

Чтобы было быстрее и без ошибок, скажи одно слово:

* **“Файлом”** — и я пришлю `editor.php` целиком с heading (как делал для button)
* **“Вставками”** — и я допишу точные куски в `renderBlocks` и `editHeadingBlock`

Как тебе удобнее?
