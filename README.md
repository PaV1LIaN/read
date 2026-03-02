Дальше логичнее всего сделать **Card-блок** (карточка/плитка). Это супер-практично: из него собираются “преимущества”, “тарифы”, “команда”, “кейсы”, и т.д. А потом на базе Card легко собрать “Cards grid” (3 карточки в ряд) уже как отдельный блок.

Сделаем тип блока: `card`

### `content`:

```json
{
  "title": "Заголовок",
  "text": "Описание...",
  "imageFileId": 123,   // опционально (из файлов сайта)
  "buttonText": "Подробнее", // опционально
  "buttonUrl": "/local/..."  // опционально
}
```

---

# 1) `api.php` — добавить `card`

## 1.1 Добавь тип в список `block.create`

В проверке типов:

```php
if (!in_array($type, ['text','image','button','heading','columns2','gallery','spacer','card'], true)) { ... }
```

## 1.2 `block.create`: ветка `card`

Добавь в формирование `$content`:

```php
} elseif ($type === 'card') {
    $title = trim((string)($_POST['title'] ?? ''));
    $text  = (string)($_POST['text'] ?? '');

    $imageFileId = (int)($_POST['imageFileId'] ?? 0);

    $buttonText = trim((string)($_POST['buttonText'] ?? ''));
    $buttonUrl  = trim((string)($_POST['buttonUrl'] ?? ''));

    if ($title === '') {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'TITLE_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    // картинка опционально, но если задана — должна быть в папке сайта
    if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
        exit;
    }

    // кнопка опционально, но если задан URL — проверим формат
    if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $content = [
        'title' => $title,
        'text' => $text,
        'imageFileId' => $imageFileId,
        'buttonText' => $buttonText,
        'buttonUrl' => $buttonUrl,
    ];
}
```

## 1.3 `block.update`: ветка `card`

```php
} elseif ($type === 'card') {
    $title = trim((string)($_POST['title'] ?? ''));
    $text  = (string)($_POST['text'] ?? '');

    $imageFileId = (int)($_POST['imageFileId'] ?? 0);

    $buttonText = trim((string)($_POST['buttonText'] ?? ''));
    $buttonUrl  = trim((string)($_POST['buttonUrl'] ?? ''));

    if ($title === '') {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'TITLE_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $b['content']['title'] = $title;
    $b['content']['text'] = $text;
    $b['content']['imageFileId'] = $imageFileId;
    $b['content']['buttonText'] = $buttonText;
    $b['content']['buttonUrl'] = $buttonUrl;
}
```

---

# 2) `editor.php` — патч (как обычно поверх твоего текущего)

## 2.1 Кнопка

Добавь в панель:

```html
<button class="ui-btn ui-btn-primary" id="btnAddCard">+ Card</button>
```

## 2.2 JS переменная

```js
const btnAddCard = document.getElementById('btnAddCard');
```

## 2.3 renderBlocks: отображение card

Перед unknown добавь:

```js
if (type === 'card') {
  const title = (b.content && typeof b.content.title === 'string') ? b.content.title : '';
  const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
  const imageFileId = (b.content && b.content.imageFileId) ? parseInt(b.content.imageFileId, 10) : 0;
  const buttonText = (b.content && typeof b.content.buttonText === 'string') ? b.content.buttonText : '';
  const buttonUrl = (b.content && typeof b.content.buttonUrl === 'string') ? b.content.buttonUrl : '';

  const img = imageFileId ? `<div class="imgPrev"><img src="${fileDownloadUrl(imageFileId)}" alt=""></div>` : '';

  return `
    <div class="block">
      <div class="row">
        <div><b>#${id}</b> <span class="muted">(card | sort ${sort})</span></div>
        <div class="btns">
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-card-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        </div>
      </div>
      <div style="margin-top:10px;">
        <div style="font-weight:700;">${BX.util.htmlspecialchars(title)}</div>
        <div class="muted" style="margin-top:6px; white-space:pre-wrap;">${BX.util.htmlspecialchars(text)}</div>
        ${img}
        ${buttonUrl ? `<a class="${btnClass('secondary')}" href="${BX.util.htmlspecialchars(buttonUrl)}" target="_blank" rel="noopener noreferrer">${BX.util.htmlspecialchars(buttonText || 'Открыть')}</a>` : ''}
      </div>
    </div>
  `;
}
```

## 2.4 addCardBlock() / editCardBlock(id)

Добавь функции:

```js
async function openCardDialog(mode, blockId, current) {
  const curTitle = current?.title || '';
  const curText = current?.text || '';
  const curImage = current?.imageFileId ? parseInt(current.imageFileId, 10) : 0;
  const curBtnText = current?.buttonText || '';
  const curBtnUrl = current?.buttonUrl || '';

  BX.UI.Dialogs.MessageBox.show({
    title: mode === 'edit' ? ('Редактировать Card #' + blockId) : 'Новый Card блок',
    message: `
      <div>
        <div class="field">
          <label>Заголовок</label>
          <input id="c_title" class="input" value="${BX.util.htmlspecialchars(curTitle)}">
        </div>
        <div class="field">
          <label>Текст</label>
          <textarea id="c_text" class="input" style="height:120px;">${BX.util.htmlspecialchars(curText)}</textarea>
        </div>

        <div class="field">
          <label>Картинка (из файлов сайта, опционально)</label>
          <select id="c_img" class="input"><option value="">Загрузка списка...</option></select>
        </div>
        <div id="c_img_prev" class="imgPrev" style="display:${curImage? 'block':'none'};">
          <img id="c_img_prev_img" src="${curImage ? fileDownloadUrl(curImage) : ''}" alt="">
        </div>

        <div class="field">
          <label>Текст кнопки (опционально)</label>
          <input id="c_btn_text" class="input" value="${BX.util.htmlspecialchars(curBtnText)}" placeholder="например: Подробнее">
        </div>
        <div class="field">
          <label>URL кнопки (опционально)</label>
          <input id="c_btn_url" class="input" value="${BX.util.htmlspecialchars(curBtnUrl)}" placeholder="https://... или /local/...">
        </div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb) {
      const title = (document.getElementById('c_title')?.value || '').trim();
      const text = (document.getElementById('c_text')?.value || '');
      const imageFileId = parseInt(document.getElementById('c_img')?.value || '0', 10);
      const buttonText = (document.getElementById('c_btn_text')?.value || '').trim();
      const buttonUrl = (document.getElementById('c_btn_url')?.value || '').trim();

      if (!title) { BX.UI.Notification.Center.notify({ content:'Введите заголовок' }); return; }

      const payload = { title, text, imageFileId, buttonText, buttonUrl };
      const call = (mode === 'edit')
        ? api('block.update', Object.assign({ id: blockId }, payload))
        : api('block.create', Object.assign({ pageId, type:'card' }, payload));

      call.then(res => {
        if (!res || res.ok !== true) { BX.UI.Notification.Center.notify({ content:'Не удалось сохранить card' }); return; }
        BX.UI.Notification.Center.notify({ content: mode === 'edit' ? 'Сохранено' : 'Card создан' });
        mb.close(); loadBlocks();
      }).catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка запроса card' }));
    }
  });

  setTimeout(async () => {
    const sel = document.getElementById('c_img');
    const prevWrap = document.getElementById('c_img_prev');
    const prevImg = document.getElementById('c_img_prev_img');
    if (!sel || !prevWrap || !prevImg) return;

    try {
      const files = await getFilesForSite();
      sel.innerHTML = '<option value="0">— без картинки —</option>' + files.map(f => {
        const s = (parseInt(f.id,10) === curImage) ? 'selected' : '';
        return `<option value="${f.id}" ${s}>${BX.util.htmlspecialchars(f.name)} (id ${f.id})</option>`;
      }).join('');

      const updatePrev = () => {
        const fid = parseInt(sel.value || '0', 10);
        if (!fid) { prevWrap.style.display = 'none'; prevImg.src = ''; return; }
        prevWrap.style.display = 'block';
        prevImg.src = fileDownloadUrl(fid);
      };
      sel.addEventListener('change', updatePrev);
      updatePrev();
    } catch (e) {
      sel.innerHTML = '<option value="0">Ошибка загрузки файлов</option>';
    }
  }, 0);
}

function addCardBlock() { openCardDialog('create', 0, null); }

function editCardBlock(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
    openCardDialog('edit', id, blk?.content || null);
  });
}
```

## 2.5 Подключи клики

В общий click handler:

```js
const ec = e.target.closest('[data-edit-card-id]');
if (ec) { editCardBlock(parseInt(ec.getAttribute('data-edit-card-id'), 10)); return; }
```

И кнопка:

```js
btnAddCard.addEventListener('click', addCardBlock);
```

---

# 3) `view.php` — рендер card

## 3.1 CSS

В `<style>` добавь:

```css
.cardBlock {
  margin-top:14px;
  background:#fff;
  border:1px solid #eee;
  border-radius:16px;
  padding:14px;
}
.cardBlock img {
  width:100%;
  height:auto;
  display:block;
  border-radius:14px;
  border:1px solid #eee;
  margin-top:10px;
}
.cardTitle { font-weight:700; font-size:18px; }
.cardText { margin-top:8px; color:#333; line-height:1.6; white-space:pre-wrap; }
.cardBtn { display:inline-block; margin-top:10px; padding:10px 14px; border-radius:12px; border:1px solid #e5e7ea; text-decoration:none; }
```

## 3.2 В цикле блоков добавь:

```php
<?php if ($type === 'card'): ?>
  <?php
    $title = (string)($b['content']['title'] ?? '');
    $text  = (string)($b['content']['text'] ?? '');
    $imgId = (int)($b['content']['imageFileId'] ?? 0);
    $btnText = trim((string)($b['content']['buttonText'] ?? ''));
    $btnUrl  = trim((string)($b['content']['buttonUrl'] ?? ''));
  ?>
  <div class="cardBlock">
    <div class="cardTitle"><?=h($title)?></div>
    <?php if ($text !== ''): ?><div class="cardText"><?=nl2br(h($text))?></div><?php endif; ?>

    <?php if ($imgId > 0): ?>
      <img src="<?=h(downloadUrl($siteId, $imgId))?>" alt="">
    <?php endif; ?>

    <?php if ($btnUrl !== ''): ?>
      <a class="cardBtn" href="<?=h($btnUrl)?>" <?= preg_match('~^https?://~i', $btnUrl) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
        <?=h($btnText !== '' ? $btnText : 'Открыть')?>
      </a>
    <?php endif; ?>
  </div>
<?php endif; ?>
```

---

## Проверка

* Создай Card без картинки и без кнопки → просто заголовок+текст
* Добавь картинку → появляется
* Добавь кнопку → появляется, внешняя ссылка открывается в новой вкладке

---

Если card встанет — следующий шаг: **Cards grid** (один блок, который содержит массив карточек 2/3/4 в ряд). Это уже прям “тильда-шаблоны”.
