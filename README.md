Ок, тогда следующий шаг без rewrite, но очень полезный для “конструктора”: **шаблоны/секции (Sections Library)** — чтобы добавлять не “один блок”, а готовую секцию из нескольких блоков (как в Tilda: “обложка”, “о нас”, “преимущества”, “галерея + текст”).

Сделаем минимально и быстро:

* хранить шаблоны в файле `/upload/sitebuilder/templates.json`
* API:

  * `template.list`
  * `template.createFromPage` (сохранить текущую страницу как шаблон)
  * `template.applyToPage` (вставить шаблон в страницу: либо “в конец”, либо “заменить страницу”)
* UI в editor.php: две кнопки

  * **“Сохранить как шаблон”**
  * **“Вставить шаблон”**

---

# 1) api.php — добавляем templates.json

## 1.1 storage функции

Добавь рядом с другими `sb_read_*`:

```php
function sb_read_templates(): array { return sb_read_json_file('templates.json'); }
function sb_write_templates(array $templates): void { sb_write_json_file('templates.json', $templates, 'Cannot open templates.json'); }
```

---

# 2) api.php — actions для шаблонов

## 2.1 template.list

```php
if ($action === 'template.list') {
    $tpl = sb_read_templates();
    usort($tpl, fn($a,$b) => strcmp((string)($b['createdAt'] ?? ''), (string)($a['createdAt'] ?? '')));
    echo json_encode(['ok'=>true,'templates'=>$tpl], JSON_UNESCAPED_UNICODE);
    exit;
}
```

## 2.2 template.createFromPage

Сохраняем *снимок* блоков страницы (только `type/content/sort`), без id.

```php
if ($action === 'template.createFromPage') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);
    $name = trim((string)($_POST['name'] ?? ''));

    if ($siteId<=0 || $pageId<=0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_PAGE_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($name==='') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $blocks = sb_read_blocks();
    $pageBlocks = array_values(array_filter($blocks, fn($b)=> (int)($b['pageId'] ?? 0) === $pageId));
    usort($pageBlocks, fn($a,$b)=> (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    $snapshot = [];
    foreach ($pageBlocks as $b) {
        $snapshot[] = [
            'type' => (string)($b['type'] ?? 'text'),
            'sort' => (int)($b['sort'] ?? 500),
            'content' => is_array($b['content'] ?? null) ? $b['content'] : [],
        ];
    }

    $tpl = sb_read_templates();
    $maxId = 0; foreach ($tpl as $t) $maxId = max($maxId, (int)($t['id'] ?? 0));
    $id = $maxId + 1;

    $record = [
        'id' => $id,
        'name' => $name,
        'createdAt' => date('c'),
        'createdBy' => (int)$USER->GetID(),
        'blocks' => $snapshot,
    ];

    $tpl[] = $record;
    sb_write_templates($tpl);

    echo json_encode(['ok'=>true,'template'=>$record], JSON_UNESCAPED_UNICODE);
    exit;
}
```

## 2.3 template.applyToPage

Два режима:

* `mode=append` (добавить в конец)
* `mode=replace` (стереть старые блоки и вставить шаблон)

```php
if ($action === 'template.applyToPage') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);
    $tplId  = (int)($_POST['templateId'] ?? 0);
    $mode   = (string)($_POST['mode'] ?? 'append');

    if ($siteId<=0 || $pageId<=0 || $tplId<=0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'PARAMS_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($mode !== 'append' && $mode !== 'replace') $mode = 'append';

    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $tplAll = sb_read_templates();
    $tpl = null;
    foreach ($tplAll as $t) if ((int)($t['id'] ?? 0) === $tplId) { $tpl = $t; break; }
    if (!$tpl) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'TEMPLATE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $blocks = sb_read_blocks();

    // очистка старых блоков
    if ($mode === 'replace') {
        $blocks = array_values(array_filter($blocks, fn($b)=> (int)($b['pageId'] ?? 0) !== $pageId));
    }

    // вычислим новые id и sort
    $maxId = 0; foreach ($blocks as $b) $maxId = max($maxId, (int)($b['id'] ?? 0));
    $nextId = $maxId + 1;

    $maxSort = 0;
    foreach ($blocks as $b) if ((int)($b['pageId'] ?? 0) === $pageId) $maxSort = max($maxSort, (int)($b['sort'] ?? 0));
    $baseSort = ($mode === 'append') ? ($maxSort + 10) : 10;

    $tplBlocks = is_array($tpl['blocks'] ?? null) ? $tpl['blocks'] : [];
    $i = 0;

    foreach ($tplBlocks as $tb) {
        if (!is_array($tb)) continue;
        $type = (string)($tb['type'] ?? 'text');
        $content = is_array($tb['content'] ?? null) ? $tb['content'] : [];

        $blocks[] = [
            'id' => $nextId++,
            'pageId' => $pageId,
            'type' => $type,
            'sort' => $baseSort + ($i * 10),
            'content' => $content,
            'createdBy' => (int)$USER->GetID(),
            'createdAt' => date('c'),
            'updatedAt' => date('c'),
        ];
        $i++;
    }

    sb_write_blocks($blocks);
    echo json_encode(['ok'=>true,'added'=>$i], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

# 3) editor.php — минимальный UI

В верхней панели рядом с “Открыть просмотр” добавь 2 кнопки:

```html
<a class="ui-btn ui-btn-light" href="javascript:void(0)" id="btnSaveTemplate">Сохранить как шаблон</a>
<a class="ui-btn ui-btn-light" href="javascript:void(0)" id="btnApplyTemplate">Вставить шаблон</a>
```

В JS добавь:

```js
const btnSaveTemplate = document.getElementById('btnSaveTemplate');
const btnApplyTemplate = document.getElementById('btnApplyTemplate');

function saveTemplateFromPage(){
  BX.UI.Dialogs.MessageBox.show({
    title:'Сохранить как шаблон',
    message:`<div class="field"><label>Название шаблона</label><input id="tpl_name" class="input" placeholder="например: Обложка + преимущества"></div>`,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      const name = (document.getElementById('tpl_name')?.value || '').trim();
      if(!name){ notify('Введите название'); return; }
      api('template.createFromPage', { siteId, pageId, name }).then(r=>{
        if(!r || r.ok!==true){ notify('Не удалось сохранить шаблон'); return; }
        notify('Шаблон сохранён');
        mb.close();
      }).catch(()=>notify('Ошибка template.createFromPage'));
    }
  });
}

async function applyTemplateToPage(){
  let res;
  try { res = await api('template.list', {}); } catch(e){ notify('Ошибка template.list'); return; }
  if(!res || res.ok!==true){ notify('Не удалось получить шаблоны'); return; }
  const list = res.templates || [];
  if(!list.length){ notify('Шаблонов нет. Сначала сохрани один.'); return; }

  const opts = list.map(t=>`<option value="${t.id}">${BX.util.htmlspecialchars(t.name)} (id ${t.id})</option>`).join('');
  BX.UI.Dialogs.MessageBox.show({
    title:'Вставить шаблон',
    message:`
      <div class="field">
        <label>Шаблон</label>
        <select id="tpl_id" class="input">${opts}</select>
      </div>
      <div class="field">
        <label>Режим</label>
        <select id="tpl_mode" class="input">
          <option value="append">Добавить в конец</option>
          <option value="replace">Заменить страницу</option>
        </select>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      const templateId = parseInt(document.getElementById('tpl_id')?.value || '0', 10);
      const mode = document.getElementById('tpl_mode')?.value || 'append';
      if(!templateId){ notify('Выбери шаблон'); return; }

      api('template.applyToPage', { siteId, pageId, templateId, mode }).then(r=>{
        if(!r || r.ok!==true){ notify('Не удалось применить шаблон'); return; }
        notify('Готово: добавлено блоков ' + (r.added || 0));
        mb.close();
        loadBlocks();
      }).catch(()=>notify('Ошибка template.applyToPage'));
    }
  });
}

btnSaveTemplate.addEventListener('click', saveTemplateFromPage);
btnApplyTemplate.addEventListener('click', applyTemplateToPage);
```

---

# 4) Что проверить

1. На странице с блоками нажми **“Сохранить как шаблон”** → задай имя → `templates.json` появится, ok=true
2. Создай новую страницу (пустую) → **“Вставить шаблон”** → append → блоки появились
3. replace → старые блоки удаляются, вставляются из шаблона

---

Если это устроит — следующим шагом сделаем **“Каталог секций” прямо в editor**: карточки шаблонов с превью (минимальное), плюс “Удалить шаблон / Переименовать”.
