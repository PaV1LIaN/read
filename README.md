([Past chat][1])([Past chat][1])([Past chat][1])([Past chat][1])

Да, **часть уже сделана**: у тебя есть `site.get`, и `view.php` уже умеет **читать** `topMenuId` и (после “пункта 3”) — `site.settings` (containerWidth/accent/logoFileId).

Но **мы ещё не делали главного** для настроек:

1. **страницу** `/local/sitebuilder/settings.php` (UI)
2. **API** `site.update` (сохранение в `sites.json`)

Это следующий шаг.

---

## 0) Мини-проверка по твоему `view.php` (важно)

Ты вставил блок с:

```php
$settings = ...; $containerWidth=...; $accent=...; $logoFileId=...
```

Это правильно **по месту** (после `$topMenuId` и до CSS/рендера).

Но проверь, что **CSS реально использует** ширину контейнера:

* у тебя сейчас `.container { max-width: var(--container); }`, а `--container` захардкожен.
* должно быть **так**:

```css
.container{
  max-width: var(--sb-container);
  margin: 0 auto;
  padding: 0 16px;
}
```

И чтобы акцент гарантированно применялся везде — хорошо, что ты используешь `--sb-accent`.

---

## 1) Добавляем `site.update` в `api.php`

### Куда вставлять

В `api.php` — в блоке `/** -------------------- SITES -------------------- */`, рядом с `site.get / site.create / site.setHome`.

### 1.1. Добавь хелпер “admin+”

Рядом с другими `sb_require_*`:

```php
function sb_require_admin(int $siteId): void { sb_require_site_role($siteId, 3); } // ADMIN/OWNER
```

### 1.2. Добавь action `site.update`

```php
if ($action === 'site.update') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    // для серьёзного проекта: только ADMIN/OWNER
    sb_require_admin($siteId);

    $sites = sb_read_sites();
    $idx = null;
    foreach ($sites as $i => $s) {
        if ((int)($s['id'] ?? 0) === $siteId) { $idx = $i; break; }
    }
    if ($idx === null) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $site = $sites[$idx];

    // --- name ---
    if (array_key_exists('name', $_POST)) {
        $name = trim((string)$_POST['name']);
        if ($name === '') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
        $site['name'] = $name;
    }

    // --- slug ---
    if (array_key_exists('slug', $_POST)) {
        $slugIn = trim((string)$_POST['slug']);
        $baseName = (string)($site['name'] ?? ('site-'.$siteId));
        $newSlug = ($slugIn === '') ? sb_slugify($baseName) : sb_slugify($slugIn);

        // уникальность slug среди сайтов
        $existing = [];
        foreach ($sites as $s) {
            if ((int)($s['id'] ?? 0) === $siteId) continue;
            $existing[] = (string)($s['slug'] ?? '');
        }
        $base = $newSlug; $k = 2;
        while (in_array($newSlug, $existing, true)) { $newSlug = $base.'-'.$k; $k++; }

        $site['slug'] = $newSlug;
    }

    // --- settings ---
    $settings = (isset($site['settings']) && is_array($site['settings'])) ? $site['settings'] : [];

    if (array_key_exists('containerWidth', $_POST)) {
        $w = (int)$_POST['containerWidth'];
        if ($w < 900) $w = 900;
        if ($w > 1600) $w = 1600;
        $settings['containerWidth'] = $w;
    }

    if (array_key_exists('accent', $_POST)) {
        $accent = trim((string)$_POST['accent']);
        if (!preg_match('~^#[0-9a-fA-F]{6}$~', $accent)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'ACCENT_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        $settings['accent'] = strtoupper($accent);
    }

    if (array_key_exists('logoFileId', $_POST)) {
        $logoFileId = (int)$_POST['logoFileId'];
        if ($logoFileId <= 0) {
            $settings['logoFileId'] = 0;
        } else {
            // файл должен лежать в папке сайта на диске
            if (!sb_disk_file_belongs_to_site($siteId, $logoFileId)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'LOGO_NOT_IN_SITE_FOLDER','fileId'=>$logoFileId], JSON_UNESCAPED_UNICODE);
                exit;
            }
            $settings['logoFileId'] = $logoFileId;
        }
    }

    $site['settings'] = $settings;

    // --- homePageId (optional) ---
    if (array_key_exists('homePageId', $_POST)) {
        $homePageId = (int)$_POST['homePageId'];
        if ($homePageId <= 0) {
            $site['homePageId'] = 0;
        } else {
            $p = sb_find_page($homePageId);
            if (!$p || (int)($p['siteId'] ?? 0) !== $siteId) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'HOME_PAGE_NOT_IN_SITE'], JSON_UNESCAPED_UNICODE);
                exit;
            }
            $site['homePageId'] = $homePageId;
        }
    }

    // --- topMenuId (optional) ---
    if (array_key_exists('topMenuId', $_POST)) {
        $topMenuId = (int)$_POST['topMenuId'];
        if ($topMenuId <= 0) {
            $site['topMenuId'] = 0;
        } else {
            // проверим, что такое меню есть у сайта
            $menusAll = sb_read_menus();
            $exists = false;
            foreach ($menusAll as $rec) {
                if ((int)($rec['siteId'] ?? 0) !== $siteId) continue;
                $menus = $rec['menus'] ?? [];
                if (!is_array($menus)) break;
                foreach ($menus as $m) {
                    if ((int)($m['id'] ?? 0) === $topMenuId) { $exists = true; break; }
                }
                break;
            }
            if (!$exists) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'TOP_MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
                exit;
            }
            $site['topMenuId'] = $topMenuId;
        }
    }

    $site['updatedAt'] = date('c');
    $site['updatedBy'] = (int)$USER->GetID();

    $sites[$idx] = $site;
    sb_write_sites($sites);

    echo json_encode(['ok'=>true,'site'=>$site], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

## 2) Готовый `settings.php`

Создай файл: `/local/sitebuilder/settings.php`

```php
<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

global $USER, $APPLICATION;

if (!$USER->IsAuthorized()) {
    LocalRedirect('/auth/');
}

header('Content-Type: text/html; charset=UTF-8');

\Bitrix\Main\UI\Extension::load([
    'main.core',
    'ui.buttons',
    'ui.dialogs.messagebox',
    'ui.notification',
]);

$siteId = (int)($_GET['siteId'] ?? 0);
?>
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Настройки сайта</title>
  <?php $APPLICATION->ShowHead(); ?>
  <style>
    body{margin:0;font-family:Arial,sans-serif;background:#f6f7f8;color:#111;}
    .wrap{max-width:1100px;margin:0 auto;padding:22px;}
    .top{
      background:#fff;border:1px solid #e5e7ea;border-radius:16px;
      padding:14px;display:flex;gap:10px;align-items:center;flex-wrap:wrap;
    }
    .title{font-weight:800;font-size:18px;}
    .muted{color:#6a737f;font-size:12px;}
    code{background:#f3f4f6;padding:2px 6px;border-radius:6px;}
    .sp{flex:1;}
    .card{
      margin-top:14px;background:#fff;border:1px solid #e5e7ea;border-radius:16px;
      padding:16px;
    }
    .grid{display:grid;gap:12px;margin-top:12px;}
    @media(min-width:860px){.grid{grid-template-columns:1fr 1fr;}}
    .field label{display:block;font-size:12px;color:#6a737f;margin-bottom:4px;}
    .input{
      width:100%;padding:9px 10px;border:1px solid #d0d7de;border-radius:10px;
      box-sizing:border-box;background:#fff;
    }
    .row{display:flex;gap:10px;align-items:center;flex-wrap:wrap;}
    .logoPrev{
      width:44px;height:44px;border-radius:12px;border:1px solid #e5e7ea;background:#fafafa;
      display:flex;align-items:center;justify-content:center;overflow:hidden;
    }
    .logoPrev img{width:100%;height:100%;object-fit:cover;display:block;}
    .hint{margin-top:8px;font-size:12px;color:#6a737f;}
    .sep{height:1px;background:#eef0f2;margin:14px 0;}
  </style>
</head>
<body>
<div class="wrap">

  <div class="top">
    <a class="ui-btn ui-btn-light ui-btn-xs" href="/local/sitebuilder/index.php">← Назад</a>
    <div>
      <div class="title">Настройки сайта</div>
      <div class="muted">siteId: <code><?= (int)$siteId ?></code></div>
    </div>
    <div class="sp"></div>
    <a class="ui-btn ui-btn-light ui-btn-xs" id="btnOpenMenu" href="/local/sitebuilder/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
    <a class="ui-btn ui-btn-light ui-btn-xs" id="btnOpenFiles" href="/local/sitebuilder/files.php?siteId=<?= (int)$siteId ?>" target="_blank">Файлы</a>
    <button class="ui-btn ui-btn-primary" id="btnSave">Сохранить</button>
  </div>

  <div class="card">
    <div class="muted">Тут настраиваем внешний вид и базовые параметры сайта. Сохранение доступно ADMIN/OWNER.</div>

    <div class="grid">
      <div class="field">
        <label>Название сайта</label>
        <input class="input" id="f_name" placeholder="Например: Лаборатория" />
      </div>

      <div class="field">
        <label>Slug (если пусто — пересчитаем)</label>
        <input class="input" id="f_slug" placeholder="lab" />
      </div>

      <div class="field">
        <label>Ширина контейнера (900–1600)</label>
        <div class="row">
          <input class="input" style="flex:1;min-width:180px;" type="range" min="900" max="1600" step="10" id="f_containerRange">
          <input class="input" style="width:140px;" type="number" min="900" max="1600" step="10" id="f_containerWidth">
        </div>
        <div class="hint">Применяется в <code>view.php</code> как <code>--sb-container</code>.</div>
      </div>

      <div class="field">
        <label>Accent (цвет)</label>
        <div class="row">
          <input class="input" style="width:90px;padding:0;height:40px;" type="color" id="f_accentColor" value="#2563eb">
          <input class="input" style="flex:1;min-width:180px;" id="f_accentText" placeholder="#2563eb">
        </div>
        <div class="hint">Формат строго <code>#RRGGBB</code>.</div>
      </div>
    </div>

    <div class="sep"></div>

    <div class="grid">
      <div class="field">
        <label>Логотип (файл из папки сайта)</label>
        <div class="row">
          <div class="logoPrev" id="logoPrev">SB</div>
          <select class="input" id="f_logoFile">
            <option value="0">— Без логотипа —</option>
          </select>
        </div>
        <div class="hint">Файлы берём из <code>files.php</code> (Disk папка сайта).</div>
      </div>

      <div class="field">
        <label>Домашняя страница</label>
        <select class="input" id="f_homePage">
          <option value="0">— Не задана —</option>
        </select>
        <div class="hint">Если задано — дальше можно будет сделать <code>/public.php?siteId=...</code> с редиректом на home.</div>
      </div>

      <div class="field">
        <label>Верхнее меню</label>
        <select class="input" id="f_topMenu">
          <option value="0">— Авто (первое) —</option>
        </select>
        <div class="hint">То же самое, что кнопка “Сделать верхним” в <code>menu.php</code>.</div>
      </div>
    </div>

  </div>

</div>

<script>
BX.ready(function(){
  const siteId = <?= (int)$siteId ?>;

  const $ = (id)=>document.getElementById(id);
  const btnSave = $('btnSave');

  function notify(msg){ BX.UI.Notification.Center.notify({content: msg}); }

  function api(action, data){
    return new Promise((resolve,reject)=>{
      BX.ajax({
        url:'/local/sitebuilder/api.php',
        method:'POST',
        dataType:'json',
        data:Object.assign({action, siteId, sessid: BX.bitrix_sessid()}, data||{}),
        onsuccess: resolve,
        onfailure: reject
      });
    });
  }

  function setLogoPreview(fileId){
    const prev = $('logoPrev');
    if (!prev) return;

    const fid = parseInt(fileId||0,10)||0;
    if (fid <= 0){
      prev.innerHTML = 'SB';
      return;
    }

    const img = document.createElement('img');
    img.src = '/local/sitebuilder/download.php?siteId=' + siteId + '&fileId=' + fid;
    img.alt = 'logo';
    prev.innerHTML = '';
    prev.appendChild(img);
  }

  function fillSelect(selectEl, items, getValue, getLabel, firstOptionKept=true){
    if (!selectEl) return;
    const keep = firstOptionKept ? selectEl.querySelector('option[value="0"]') : null;
    selectEl.innerHTML = '';
    if (keep) selectEl.appendChild(keep);

    items.forEach(it=>{
      const opt = document.createElement('option');
      opt.value = String(getValue(it));
      opt.textContent = getLabel(it);
      selectEl.appendChild(opt);
    });
  }

  async function load(){
    if (!siteId){
      notify('siteId не задан');
      return;
    }

    try{
      const [sres, pres, mres, fres] = await Promise.all([
        api('site.get'),
        api('page.list'),
        api('menu.list'),
        api('file.list')
      ]);

      if (!sres || sres.ok !== true){ notify('site.get: нет доступа'); return; }
      if (!pres || pres.ok !== true){ notify('page.list: нет доступа'); return; }
      if (!mres || mres.ok !== true){ notify('menu.list: нет доступа'); return; }
      if (!fres || fres.ok !== true){ notify('file.list: нет доступа'); return; }

      const site = sres.site || {};
      const settings = (site.settings && typeof site.settings === 'object') ? site.settings : {};

      $('f_name').value = site.name || '';
      $('f_slug').value = site.slug || '';

      const w = parseInt(settings.containerWidth || 1100,10) || 1100;
      $('f_containerWidth').value = String(w);
      $('f_containerRange').value = String(w);

      const accent = (settings.accent || '#2563eb').toString();
      $('f_accentText').value = accent;
      $('f_accentColor').value = accent;

      const logoFileId = parseInt(settings.logoFileId||0,10)||0;

      // pages
      const pages = pres.pages || [];
      fillSelect($('f_homePage'), pages, x=>x.id, x=>`#${x.id} ${x.title} (${x.slug})`);
      $('f_homePage').value = String(parseInt(site.homePageId||0,10)||0);

      // menus
      const menus = mres.menus || [];
      fillSelect($('f_topMenu'), menus, x=>x.id, x=>`#${x.id} ${x.name}`);
      $('f_topMenu').value = String(parseInt(site.topMenuId||0,10)||0);

      // files for logo
      const files = fres.files || [];
      fillSelect($('f_logoFile'), files, x=>x.id, x=>`#${x.id} ${x.name}`);
      $('f_logoFile').value = String(logoFileId);
      setLogoPreview(logoFileId);

    } catch(e){
      notify('Ошибка загрузки настроек');
    }
  }

  function bind(){
    // width sync
    $('f_containerRange').addEventListener('input', ()=>{
      $('f_containerWidth').value = $('f_containerRange').value;
    });
    $('f_containerWidth').addEventListener('input', ()=>{
      $('f_containerRange').value = $('f_containerWidth').value;
    });

    // accent sync
    $('f_accentColor').addEventListener('input', ()=>{
      $('f_accentText').value = $('f_accentColor').value;
    });
    $('f_accentText').addEventListener('input', ()=>{
      const v = $('f_accentText').value.trim();
      if (/^#[0-9a-fA-F]{6}$/.test(v)) $('f_accentColor').value = v;
    });

    $('f_logoFile').addEventListener('change', ()=>{
      setLogoPreview($('f_logoFile').value);
    });

    btnSave.addEventListener('click', async ()=>{
      const payload = {
        name: $('f_name').value.trim(),
        slug: $('f_slug').value.trim(),
        containerWidth: parseInt($('f_containerWidth').value||'1100',10) || 1100,
        accent: $('f_accentText').value.trim(),
        logoFileId: parseInt($('f_logoFile').value||'0',10) || 0,
        homePageId: parseInt($('f_homePage').value||'0',10) || 0,
        topMenuId: parseInt($('f_topMenu').value||'0',10) || 0
      };

      try{
        const r = await api('site.update', payload);
        if (!r || r.ok !== true){
          notify('Не удалось сохранить (возможно нет прав ADMIN/OWNER)');
          return;
        }
        notify('Сохранено');
      }catch(e){
        notify('Ошибка site.update');
      }
    });
  }

  bind();
  load();
});
</script>
</body>
</html>
```

---

## 3) Маленький штрих: кнопка “Настройки” в `index.php`

В карточке сайта (где у тебя кнопки “Страницы/Меню/Файлы/Доступы…”) добавь:

```html
<a class="ui-btn ui-btn-light ui-btn-xs" href="/local/sitebuilder/settings.php?siteId=${s.id}">
  Настройки
</a>
```

---

Если хочешь — следующим шагом сделаем **settings применяемыми “везде”**:

* показывать логотип в `view.php` (если `logoFileId > 0`)
* применить `--sb-container` реально (я выше указал 1 строку)
* и потом уже аккуратно подключим `public.php` к `homePageId`.

[1]: https://chatgpt.com/c/6997ed68-f66c-8395-8b8f-7fa8af7d70b9 "Создание сайта на Битрикс24"
