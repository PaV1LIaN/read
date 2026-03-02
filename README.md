Отлично. Следующий “тильда-уровень” — **Gallery (несколько картинок)**.

Сделаем тип блока: `gallery`
**content**:

```json
{
  "columns": 3,
  "images": [
    {"fileId": 123, "alt": "Фото 1"},
    {"fileId": 124, "alt": "Фото 2"}
  ]
}
```

Ниже — готовые патчи для **api.php**, **editor.php**, **view.php**.

---

# 1) `api.php` — добавить тип `gallery`

## 1.1 Разрешить `gallery` в `block.create`

В `block.create` в проверке типов добавь `gallery`:

```php
if (!in_array($type, ['text','image','button','heading','columns2','gallery'], true)) { ... }
```

## 1.2 `block.create`: ветка `gallery`

Внутри формирования `$content` добавь:

```php
} elseif ($type === 'gallery') {
    $columns = (int)($_POST['columns'] ?? 3);
    if (!in_array($columns, [2,3,4], true)) $columns = 3;

    $imagesJson = (string)($_POST['images'] ?? '[]');
    $images = json_decode($imagesJson, true);
    if (!is_array($images)) $images = [];

    // нормализуем + проверяем, что файлы лежат в папке сайта
    $clean = [];
    foreach ($images as $it) {
        if (!is_array($it)) continue;
        $fid = (int)($it['fileId'] ?? 0);
        if ($fid <= 0) continue;

        if (!sb_disk_file_belongs_to_site($siteId, $fid)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$fid], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $clean[] = [
            'fileId' => $fid,
            'alt' => (string)($it['alt'] ?? ''),
        ];
    }

    if (count($clean) === 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'IMAGES_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $content = [
        'columns' => $columns,
        'images' => $clean,
    ];
}
```

## 1.3 `block.update`: ветка `gallery`

В `block.update` добавь:

```php
} elseif ($type === 'gallery') {
    $columns = (int)($_POST['columns'] ?? 3);
    if (!in_array($columns, [2,3,4], true)) $columns = 3;

    $imagesJson = (string)($_POST['images'] ?? '[]');
    $images = json_decode($imagesJson, true);
    if (!is_array($images)) $images = [];

    $clean = [];
    foreach ($images as $it) {
        if (!is_array($it)) continue;
        $fid = (int)($it['fileId'] ?? 0);
        if ($fid <= 0) continue;

        if (!sb_disk_file_belongs_to_site($siteId, $fid)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$fid], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $clean[] = [
            'fileId' => $fid,
            'alt' => (string)($it['alt'] ?? ''),
        ];
    }

    if (count($clean) === 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'IMAGES_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $b['content']['columns'] = $columns;
    $b['content']['images'] = $clean;
}
```

---

# 2) `editor.php` — добавить Gallery (патч поверх твоего текущего)

## 2.1 CSS

В `<style>` добавь:

```css
/* gallery preview */
.galPrev { margin-top:10px; display:grid; gap:10px; }
.galPrev img { width:100%; height:auto; display:block; border-radius:10px; border:1px solid #eee; background:#fafafa; }
.galPick { margin-top:10px; max-height:260px; overflow:auto; border:1px solid #e5e7ea; border-radius:10px; padding:10px; background:#fff; }
.galPick .row { display:flex; justify-content:flex-start; gap:10px; align-items:center; margin:6px 0; }
.galPick small { color:#6a737f; }
```

## 2.2 Кнопка

В кнопки добавь:

```html
<button class="ui-btn ui-btn-primary" id="btnAddGallery">+ Gallery</button>
```

## 2.3 JS: переменная кнопки

Рядом с остальными:

```js
const btnAddGallery = document.getElementById('btnAddGallery');
```

## 2.4 В `renderBlocks()` добавь обработку `gallery`

Перед `unknown` вставь:

```js
if (type === 'gallery') {
  const columns = (b.content && b.content.columns) ? parseInt(b.content.columns, 10) : 3;
  const imgs = (b.content && Array.isArray(b.content.images)) ? b.content.images : [];

  const tpl = (columns === 2) ? '1fr 1fr' : (columns === 4) ? '1fr 1fr 1fr 1fr' : '1fr 1fr 1fr';
  const prev = imgs.map(it => {
    const fid = parseInt(it.fileId || 0, 10);
    if (!fid) return '';
    return `<img src="${fileDownloadUrl(fid)}" alt="${BX.util.htmlspecialchars(it.alt || '')}">`;
  }).join('');

  return `
    <div class="block">
      <div class="row">
        <div><b>#${id}</b> <span class="muted">(gallery | sort ${sort} | cols ${columns} | images ${imgs.length})</span></div>
        <div class="btns">
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-gallery-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        </div>
      </div>
      <div class="galPrev" style="grid-template-columns:${tpl};">${prev}</div>
    </div>
  `;
}
```

## 2.5 Функции: добавить `addGalleryBlock()` и `editGalleryBlock(id)`

Добавь **после твоих add… функций**:

```js
async function openGalleryDialog(mode, blockId, currentContent) {
  const currentCols = currentContent?.columns ? parseInt(currentContent.columns, 10) : 3;
  const currentImages = Array.isArray(currentContent?.images) ? currentContent.images : [];

  BX.UI.Dialogs.MessageBox.show({
    title: mode === 'edit' ? ('Редактировать Gallery #' + blockId) : 'Новый Gallery блок',
    message: `
      <div>
        <div class="field">
          <label>Колонки</label>
          <select id="g_cols" class="input">
            <option value="2" ${currentCols===2?'selected':''}>2</option>
            <option value="3" ${currentCols===3?'selected':''}>3</option>
            <option value="4" ${currentCols===4?'selected':''}>4</option>
          </select>
        </div>

        <div class="muted" style="margin-top:8px;">Выбери файлы из “Файлы” сайта:</div>
        <div id="g_list" class="galPick">Загрузка списка...</div>

        <div class="muted" style="margin-top:10px;">Превью:</div>
        <div id="g_prev" class="galPrev"></div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      const cols = parseInt(document.getElementById('g_cols')?.value || '3', 10);
      const list = document.getElementById('g_list');
      if (!list) return;

      const checks = list.querySelectorAll('input[type="checkbox"][data-fid]');
      const selected = [];
      checks.forEach(ch => {
        if (!ch.checked) return;
        const fid = parseInt(ch.getAttribute('data-fid'), 10);
        const altEl = list.querySelector(`input[data-alt-for="${fid}"]`);
        const alt = (altEl?.value || '').trim();
        if (fid) selected.push({ fileId: fid, alt });
      });

      if (!selected.length) {
        BX.UI.Notification.Center.notify({ content:'Выбери хотя бы 1 файл' });
        return;
      }

      const images = JSON.stringify(selected);

      const payload = { columns: cols, images };
      const call = (mode === 'edit')
        ? api('block.update', Object.assign({ id: blockId }, payload))
        : api('block.create', Object.assign({ pageId, type:'gallery' }, payload));

      call.then(res => {
        if (!res || res.ok !== true) {
          BX.UI.Notification.Center.notify({ content:'Не удалось сохранить gallery' });
          return;
        }
        BX.UI.Notification.Center.notify({ content: mode==='edit' ? 'Сохранено' : 'Gallery создан' });
        mb.close();
        loadBlocks();
      }).catch(() => BX.UI.Notification.Center.notify({ content:'Ошибка запроса gallery' }));
    }
  });

  // заполняем список файлов + превью
  setTimeout(async () => {
    const box = document.getElementById('g_list');
    const prev = document.getElementById('g_prev');
    const colsSel = document.getElementById('g_cols');
    if (!box || !prev || !colsSel) return;

    const selectedMap = {};
    currentImages.forEach(it => { selectedMap[parseInt(it.fileId,10)] = (it.alt || ''); });

    try {
      const files = await getFilesForSite();
      if (!files.length) {
        box.innerHTML = '<div class="muted">Файлов нет (загрузите в “Файлы”)</div>';
        return;
      }

      box.innerHTML = files.map(f => {
        const checked = selectedMap[f.id] !== undefined ? 'checked' : '';
        const altVal = selectedMap[f.id] !== undefined ? selectedMap[f.id] : '';
        return `
          <div class="row">
            <input type="checkbox" data-fid="${f.id}" ${checked}>
            <div style="flex:1;">
              <div><b>${BX.util.htmlspecialchars(f.name)}</b> <small>(id ${f.id})</small></div>
              <input class="input" style="margin-top:6px;" data-alt-for="${f.id}" placeholder="alt (опционально)" value="${BX.util.htmlspecialchars(altVal)}">
            </div>
          </div>
        `;
      }).join('');

      const renderPrev = () => {
        const cols = parseInt(colsSel.value || '3', 10);
        const tpl = (cols===2) ? '1fr 1fr' : (cols===4) ? '1fr 1fr 1fr 1fr' : '1fr 1fr 1fr';
        prev.style.gridTemplateColumns = tpl;

        const checks = box.querySelectorAll('input[type="checkbox"][data-fid]');
        let html = '';
        checks.forEach(ch => {
          if (!ch.checked) return;
          const fid = parseInt(ch.getAttribute('data-fid'), 10);
          if (!fid) return;
          html += `<img src="${fileDownloadUrl(fid)}" alt="">`;
        });
        prev.innerHTML = html || '<div class="muted">Ничего не выбрано</div>';
      };

      box.addEventListener('change', renderPrev);
      colsSel.addEventListener('change', renderPrev);
      renderPrev();

    } catch (e) {
      box.innerHTML = '<div class="muted">Ошибка загрузки файлов</div>';
    }
  }, 0);
}

function addGalleryBlock() { openGalleryDialog('create', 0, null); }

function editGalleryBlock(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
    openGalleryDialog('edit', id, blk?.content || null);
  });
}
```

## 2.6 Подключить клики

В `document.addEventListener('click', ...)` добавь:

```js
const eg = e.target.closest('[data-edit-gallery-id]');
if (eg) { editGalleryBlock(parseInt(eg.getAttribute('data-edit-gallery-id'), 10)); return; }
```

Внизу, где кнопки:

```js
btnAddGallery.addEventListener('click', addGalleryBlock);
```

---

# 3) `view.php` — рендер gallery

## 3.1 CSS

В `<style>` добавь:

```css
.gallery {
  margin-top: 14px;
  display: grid;
  gap: 12px;
}
.gallery img {
  width: 100%;
  height: auto;
  display: block;
  border-radius: 12px;
  border: 1px solid #eee;
  background: #fafafa;
}
@media (max-width: 768px) {
  .gallery { grid-template-columns: 1fr !important; }
}
```

## 3.2 В цикле блоков добавь:

```php
<?php if ($type === 'gallery'): ?>
  <?php
    $cols = (int)($b['content']['columns'] ?? 3);
    if (!in_array($cols, [2,3,4], true)) $cols = 3;

    $tpl = '1fr 1fr 1fr';
    if ($cols === 2) $tpl = '1fr 1fr';
    if ($cols === 4) $tpl = '1fr 1fr 1fr 1fr';

    $imgs = $b['content']['images'] ?? [];
    if (!is_array($imgs)) $imgs = [];
  ?>
  <div class="gallery" style="grid-template-columns: <?=h($tpl)?>;">
    <?php foreach ($imgs as $it): ?>
      <?php
        if (!is_array($it)) continue;
        $fid = (int)($it['fileId'] ?? 0);
        $alt = (string)($it['alt'] ?? '');
        if ($fid <= 0) continue;
      ?>
      <img src="<?= h(downloadUrl($siteId, $fid)) ?>" alt="<?= h($alt) ?>">
    <?php endforeach; ?>
  </div>
<?php endif; ?>
```

---

# Что проверить

1. В “Файлы” загрузить 3–6 картинок
2. В редакторе нажать **+ Gallery**, выбрать несколько файлов, cols=3 → OK
3. В `view.php` увидеть сетку
4. На мобилке (узкое окно) — всё становится 1 колонкой
5. “Редактировать” gallery: добавить/убрать картинки, сменить cols.

---

Если gallery ок — дальше самый полезный “лендинговый” блок: **Spacer/Divider** (контролируемые отступы) *или* **Card (иконка+заголовок+текст)**.
