Отлично — делаем **каталог секций (Templates Library) прямо в editor**: карточки шаблонов + поиск + кнопки **Вставить / Заменить / Переименовать / Удалить**.

Сделаем без превью-картинок (пока), но с полезной инфой: сколько блоков, когда создан, кем.

---

# 1) api.php — добавляем 2 actions: `template.rename`, `template.delete`

Вставь перед `UNKNOWN_ACTION`.

### `template.rename`

```php
if ($action === 'template.rename') {
    $id = (int)($_POST['id'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($name === '') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $tpl = sb_read_templates();
    $found = false;

    foreach ($tpl as &$t) {
        if ((int)($t['id'] ?? 0) === $id) {
            $t['name'] = $name;
            $t['updatedAt'] = date('c');
            $t['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($t);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'TEMPLATE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_templates($tpl);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

### `template.delete`

```php
if ($action === 'template.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $tpl = sb_read_templates();
    $before = count($tpl);
    $tpl = array_values(array_filter($tpl, fn($t) => (int)($t['id'] ?? 0) !== $id));

    if (count($tpl) === $before) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'TEMPLATE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_templates($tpl);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

# 2) editor.php — делаем “Каталог секций” кнопкой (вместо 100 новых экранов)

## 2.1 Кнопка

В top-панели добавь ещё одну:

```html
<a class="ui-btn ui-btn-light" href="javascript:void(0)" id="btnSections">Каталог секций</a>
```

## 2.2 CSS для “каталога”

В `<style>` добавь:

```css
/* sections library */
.secGrid{display:grid;gap:12px;margin-top:12px;}
@media (min-width: 820px){ .secGrid{grid-template-columns: 1fr 1fr;} }
.secCard{border:1px solid #e5e7ea;border-radius:14px;background:#fff;padding:12px;}
.secTitle{font-weight:800;}
.secMeta{color:#6a737f;font-size:12px;margin-top:4px;}
.secBtns{display:flex;gap:6px;flex-wrap:wrap;margin-top:10px;}
.secSearch{margin-top:10px;}
```

## 2.3 JS: логика каталога

Внутри `BX.ready(...)`:

### 2.3.1 Получаем кнопку

Рядом с другими `const ...`:

```js
const btnSections = document.getElementById('btnSections');
```

### 2.3.2 Функция открытия каталога

Вставь **после** `api/notify/loadBlocks` (как делал с шаблонами):

```js
async function openSectionsLibrary() {
  let res;
  try { res = await api('template.list', {}); } catch(e) { notify('Ошибка template.list'); return; }
  if (!res || res.ok !== true) { notify('Не удалось получить шаблоны'); return; }

  let templates = res.templates || [];
  if (!templates.length) { notify('Шаблонов нет. Сначала сохрани страницу как шаблон.'); return; }

  const render = (q) => {
    const query = (q || '').trim().toLowerCase();
    const filtered = templates.filter(t => (t.name || '').toLowerCase().includes(query));

    const cards = filtered.map(t => {
      const blocksCount = Array.isArray(t.blocks) ? t.blocks.length : 0;
      const createdAt = (t.createdAt || '').replace('T',' ').replace('Z','');
      return `
        <div class="secCard" data-tpl="${t.id}">
          <div class="secTitle">${BX.util.htmlspecialchars(t.name || ('Template #' + t.id))}</div>
          <div class="secMeta">id: ${t.id} • блоков: ${blocksCount} • создан: ${BX.util.htmlspecialchars(createdAt)}</div>

          <div class="secBtns">
            <button class="ui-btn ui-btn-primary ui-btn-xs" data-tpl-apply="${t.id}" data-mode="append">Вставить</button>
            <button class="ui-btn ui-btn-light ui-btn-xs" data-tpl-apply="${t.id}" data-mode="replace">Заменить</button>
            <button class="ui-btn ui-btn-light ui-btn-xs" data-tpl-rename="${t.id}">Переименовать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-tpl-delete="${t.id}">Удалить</button>
          </div>
        </div>
      `;
    }).join('');

    return `
      <div>
        <div class="secSearch">
          <input id="sec_q" class="input" placeholder="Поиск шаблонов..." value="${BX.util.htmlspecialchars(q || '')}">
        </div>
        <div class="secGrid">${cards || '<div class="muted">Ничего не найдено</div>'}</div>
      </div>
    `;
  };

  BX.UI.Dialogs.MessageBox.show({
    title: 'Каталог секций',
    message: render(''),
    buttons: BX.UI.Dialogs.MessageBoxButtons.CANCEL,
    onCancel: function(mb){ mb.close(); }
  });

  setTimeout(() => {
    const mb = document.querySelector('.bx-ui-dialogs-messagebox');
    const root = mb ? mb.querySelector('.bx-ui-dialogs-messagebox-content') : null;
    if (!root) return;

    const rerender = (q) => {
      root.innerHTML = render(q);
      bind();
    };

    const bind = () => {
      const q = document.getElementById('sec_q');
      if (q) q.oninput = () => rerender(q.value);

      root.querySelectorAll('[data-tpl-apply]').forEach(btn => {
        btn.onclick = async () => {
          const tplId = parseInt(btn.getAttribute('data-tpl-apply'), 10);
          const mode = btn.getAttribute('data-mode') || 'append';
          try {
            const r = await api('template.applyToPage', { siteId, pageId, templateId: tplId, mode });
            if (!r || r.ok !== true) { notify('Не удалось применить'); return; }
            notify('Готово: добавлено блоков ' + (r.added || 0));
            loadBlocks();
          } catch(e) { notify('Ошибка apply'); }
        };
      });

      root.querySelectorAll('[data-tpl-rename]').forEach(btn => {
        btn.onclick = () => {
          const tplId = parseInt(btn.getAttribute('data-tpl-rename'), 10);
          const cur = templates.find(x => parseInt(x.id,10) === tplId);
          BX.UI.Dialogs.MessageBox.show({
            title: 'Переименовать',
            message: `<div class="field"><label>Название</label><input id="rn_name" class="input" value="${BX.util.htmlspecialchars(cur?.name || '')}"></div>`,
            buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
            onOk: function(mb2){
              const name = (document.getElementById('rn_name')?.value || '').trim();
              if (!name) { notify('Введите название'); return; }
              api('template.rename', { id: tplId, name }).then(r=>{
                if(!r || r.ok!==true){ notify('Не удалось переименовать'); return; }
                notify('Переименовано');
                const idx = templates.findIndex(x => parseInt(x.id,10) === tplId);
                if (idx >= 0) templates[idx].name = name;
                mb2.close();
                const q = document.getElementById('sec_q')?.value || '';
                rerender(q);
              }).catch(()=>notify('Ошибка template.rename'));
            }
          });
        };
      });

      root.querySelectorAll('[data-tpl-delete]').forEach(btn => {
        btn.onclick = () => {
          const tplId = parseInt(btn.getAttribute('data-tpl-delete'), 10);
          BX.UI.Dialogs.MessageBox.show({
            title: 'Удалить шаблон?',
            message: 'Удалить навсегда?',
            buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
            onOk: function(mb2){
              api('template.delete', { id: tplId }).then(r=>{
                if(!r || r.ok!==true){ notify('Не удалось удалить'); return; }
                notify('Удалено');
                templates = templates.filter(x => parseInt(x.id,10) !== tplId);
                mb2.close();
                const q = document.getElementById('sec_q')?.value || '';
                rerender(q);
              }).catch(()=>notify('Ошибка template.delete'));
            }
          });
        };
      });
    };

    bind();
  }, 0);
}
```

### 2.3.3 Вешаем обработчик

В “buttons” секции:

```js
btnSections.addEventListener('click', openSectionsLibrary);
```

---

# 3) Проверка

1. Нажми **Каталог секций**
2. Появится список карточек
3. Вставить → блоки добавились
4. Заменить → страница перезаписалась блоками шаблона
5. Переименовать/Удалить → работает и сразу обновляет каталог

---

Если хочешь следующий апгрейд: **мини-превью секции** (не картинки, а “скелет”: heading/text/image-icons) — это красиво и быстро.
