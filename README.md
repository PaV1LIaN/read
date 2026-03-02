Отлично — следующий логичный шаг: **Cards Grid** (один блок, внутри которого несколько карточек 2/3/4 в ряд). Это реально “тильда-секция”: добавил 3 карточки — получил готовый блок “Преимущества/Тарифы”.

Сделаем тип блока: `cards`

### `content`

```json
{
  "columns": 3,
  "items": [
    { "title":"...", "text":"...", "imageFileId":123, "buttonText":"...", "buttonUrl":"..." },
    { "title":"...", "text":"..." }
  ]
}
```

---

# 1) api.php — добавляем `cards`

## 1.1 Разрешаем тип в `block.create`

В проверке типов добавь `cards`:

```php
if (!in_array($type, ['text','image','button','heading','columns2','gallery','spacer','card','cards'], true)) { ... }
```

## 1.2 `block.create` — ветка `cards`

Внутри формирования `$content` добавь:

```php
} elseif ($type === 'cards') {
    $columns = (int)($_POST['columns'] ?? 3);
    if (!in_array($columns, [2,3,4], true)) $columns = 3;

    $itemsJson = (string)($_POST['items'] ?? '[]');
    $items = json_decode($itemsJson, true);
    if (!is_array($items)) $items = [];

    $clean = [];
    foreach ($items as $it) {
        if (!is_array($it)) continue;

        $title = trim((string)($it['title'] ?? ''));
        $text  = (string)($it['text'] ?? '');

        if ($title === '') continue; // минимум — заголовок

        $imageFileId = (int)($it['imageFileId'] ?? 0);
        if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $buttonText = trim((string)($it['buttonText'] ?? ''));
        $buttonUrl  = trim((string)($it['buttonUrl'] ?? ''));

        if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $clean[] = [
            'title' => $title,
            'text' => $text,
            'imageFileId' => $imageFileId,
            'buttonText' => $buttonText,
            'buttonUrl' => $buttonUrl,
        ];
    }

    if (count($clean) === 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ITEMS_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $content = [
        'columns' => $columns,
        'items' => $clean,
    ];
}
```

## 1.3 `block.update` — ветка `cards`

Аналогично:

```php
} elseif ($type === 'cards') {
    $columns = (int)($_POST['columns'] ?? 3);
    if (!in_array($columns, [2,3,4], true)) $columns = 3;

    $itemsJson = (string)($_POST['items'] ?? '[]');
    $items = json_decode($itemsJson, true);
    if (!is_array($items)) $items = [];

    $clean = [];
    foreach ($items as $it) {
        if (!is_array($it)) continue;

        $title = trim((string)($it['title'] ?? ''));
        $text  = (string)($it['text'] ?? '');

        if ($title === '') continue;

        $imageFileId = (int)($it['imageFileId'] ?? 0);
        if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $buttonText = trim((string)($it['buttonText'] ?? ''));
        $buttonUrl  = trim((string)($it['buttonUrl'] ?? ''));

        if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $clean[] = [
            'title' => $title,
            'text' => $text,
            'imageFileId' => $imageFileId,
            'buttonText' => $buttonText,
            'buttonUrl' => $buttonUrl,
        ];
    }

    if (count($clean) === 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ITEMS_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $b['content']['columns'] = $columns;
    $b['content']['items'] = $clean;
}
```

---

# 2) view.php — рендер `cards`

## 2.1 CSS

Добавь в `<style>`:

```css
.cardsGrid {
  margin-top:14px;
  display:grid;
  gap:14px;
}
.cardsGrid .cardItem {
  background:#fff;
  border:1px solid #eee;
  border-radius:16px;
  padding:14px;
}
.cardsGrid .cardItem img{
  width:100%;
  height:auto;
  display:block;
  border-radius:14px;
  border:1px solid #eee;
  background:#fafafa;
  margin-top:10px;
}
.cardsGrid .t { font-weight:700; font-size:16px; }
.cardsGrid .d { margin-top:8px; color:#333; line-height:1.6; white-space:pre-wrap; }
.cardsGrid .a { display:inline-block; margin-top:10px; padding:10px 14px; border-radius:12px; border:1px solid #e5e7ea; text-decoration:none; }
@media (max-width: 768px) {
  .cardsGrid { grid-template-columns: 1fr !important; }
}
```

## 2.2 В цикле блоков добавь:

```php
<?php if ($type === 'cards'): ?>
  <?php
    $cols = (int)($b['content']['columns'] ?? 3);
    if (!in_array($cols, [2,3,4], true)) $cols = 3;

    $tpl = '1fr 1fr 1fr';
    if ($cols === 2) $tpl = '1fr 1fr';
    if ($cols === 4) $tpl = '1fr 1fr 1fr 1fr';

    $items = $b['content']['items'] ?? [];
    if (!is_array($items)) $items = [];
  ?>
  <div class="cardsGrid" style="grid-template-columns: <?=h($tpl)?>;">
    <?php foreach ($items as $it): ?>
      <?php
        if (!is_array($it)) continue;
        $title = (string)($it['title'] ?? '');
        if ($title === '') continue;
        $text = (string)($it['text'] ?? '');
        $imgId = (int)($it['imageFileId'] ?? 0);
        $btnText = trim((string)($it['buttonText'] ?? ''));
        $btnUrl  = trim((string)($it['buttonUrl'] ?? ''));
      ?>
      <div class="cardItem">
        <div class="t"><?=h($title)?></div>
        <?php if ($text !== ''): ?><div class="d"><?=nl2br(h($text))?></div><?php endif; ?>
        <?php if ($imgId > 0): ?><img src="<?=h(downloadUrl($siteId, $imgId))?>" alt=""><?php endif; ?>
        <?php if ($btnUrl !== ''): ?>
          <a class="a" href="<?=h($btnUrl)?>" <?= preg_match('~^https?://~i', $btnUrl) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
            <?=h($btnText !== '' ? $btnText : 'Открыть')?>
          </a>
        <?php endif; ?>
      </div>
    <?php endforeach; ?>
  </div>
<?php endif; ?>
```

---

# 3) editor.php — как делаем проще (без огромного конструктора)

Чтобы не городить UI на 200 строк, сделаем первый вариант минимально:

* “Cards” редактируем **через JSON textarea** (позже сделаем красивый конструктор)
* Это быстрый путь и надежный.

## 3.1 Кнопка

Добавь:

```html
<button class="ui-btn ui-btn-primary" id="btnAddCards">+ Cards</button>
```

## 3.2 JS переменная

```js
const btnAddCards = document.getElementById('btnAddCards');
```

## 3.3 renderBlocks (превью)

Добавь обработку `cards` (перед unknown):

```js
if (type === 'cards') {
  const columns = (b.content && b.content.columns) ? parseInt(b.content.columns, 10) : 3;
  const items = (b.content && Array.isArray(b.content.items)) ? b.content.items : [];
  return `
    <div class="block">
      <div class="row">
        <div><b>#${id}</b> <span class="muted">(cards | sort ${sort} | cols ${columns} | items ${items.length})</span></div>
        <div class="btns">
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cards-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        </div>
      </div>
      <div class="muted" style="margin-top:8px;">Пока редактирование через JSON (быстро). Потом сделаем визуальный конструктор.</div>
      <pre>${BX.util.htmlspecialchars(JSON.stringify({columns, items}, null, 2))}</pre>
    </div>
  `;
}
```

## 3.4 add/edit через JSON textarea

Добавь функции:

```js
function openCardsJsonDialog(mode, blockId, currentContent) {
  const cur = currentContent && typeof currentContent === 'object' ? currentContent : { columns: 3, items: [] };
  const json = JSON.stringify(cur, null, 2);

  BX.UI.Dialogs.MessageBox.show({
    title: mode === 'edit' ? ('Редактировать Cards #' + blockId) : 'Новый Cards блок',
    message: `
      <div class="muted">Формат: { columns: 2|3|4, items: [ {title,text,imageFileId,buttonText,buttonUrl}, ... ] }</div>
      <textarea id="cards_json" class="input" style="height:260px; font-family:monospace;">${BX.util.htmlspecialchars(json)}</textarea>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      let obj = null;
      try {
        obj = JSON.parse(document.getElementById('cards_json')?.value || '');
      } catch (e) {
        notify('JSON не валиден');
        return;
      }

      const columns = parseInt(obj.columns || 3, 10);
      if (![2,3,4].includes(columns)) { notify('columns должен быть 2/3/4'); return; }

      const items = Array.isArray(obj.items) ? obj.items : [];
      if (!items.length) { notify('items пустой'); return; }

      const payload = { columns, items: JSON.stringify(items) };

      const call = (mode === 'edit')
        ? api('block.update', Object.assign({ id: blockId }, payload))
        : api('block.create', Object.assign({ pageId, type:'cards' }, payload));

      call.then(res => {
        if (!res || res.ok !== true) { notify('Не удалось сохранить cards'); return; }
        notify(mode === 'edit' ? 'Сохранено' : 'Cards создан');
        mb.close(); loadBlocks();
      }).catch(() => notify('Ошибка запроса cards'));
    }
  });
}

function addCardsBlock() { openCardsJsonDialog('create', 0, { columns: 3, items: [
  { title: 'Преимущество 1', text: 'Короткое описание' },
  { title: 'Преимущество 2', text: 'Короткое описание' },
  { title: 'Преимущество 3', text: 'Короткое описание' }
]}); }

function editCardsBlock(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
    openCardsJsonDialog('edit', id, blk?.content || null);
  });
}
```

## 3.5 Подключи клики

В click handler:

```js
const ecards = e.target.closest('[data-edit-cards-id]');
if (ecards) { editCardsBlock(parseInt(ecards.getAttribute('data-edit-cards-id'), 10)); return; }
```

И кнопку:

```js
btnAddCards.addEventListener('click', addCardsBlock);
```

---

## Что проверить

1. “+ Cards” → сразу создается пример из 3 карточек
2. В `view.php` сетка отображается в 3 колонки
3. Отредактируй JSON: поставь `columns: 2` и измени текст — сохрани
4. Если указал `imageFileId`, картинка должна быть из папки сайта (иначе API ругнется)

---

Если согласен — дальше делаем **визуальный конструктор Cards** (без JSON), где можно:

* добавить/удалить карточку,
* выбрать картинку из файлов,
* двигать карточки вверх/вниз.
