Отлично. Тогда следующий шаг — **`settings.php` + `api.site.update`** (и чуть-чуть дополним структуру сайта в `sites.json`, но без ломания обратной совместимости).

Ниже даю:

1. **что именно будем хранить в site**
2. **готовый код для `api.site.update`** (вставка в `api.php`)
3. **готовый `settings.php` целиком** (страница настроек сайта)

---

## 1) Что добавим в `sites.json` (мягко, без обязательности)

Мы добавим поле `settings` (если его нет — считаем дефолт):

```json
{
  "settings": {
    "containerWidth": 1100,
    "accent": "#2563eb",
    "logoFileId": 0
  }
}
```

Ничего старого не ломает: если `settings` отсутствует — `view.php` просто использует свои дефолты (у тебя сейчас так и есть).

---

## 2) Вставь в `api.php`: `site.update`

Вставь этот блок **рядом с другими site.* экшенами** (логично: после `site.get` или после `site.setHome` — не важно, главное ДО финального `UNKNOWN_ACTION`).

```php
// site.update (EDITOR+)
if ($action === 'site.update') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_editor($siteId);

    $name = trim((string)($_POST['name'] ?? ''));
    $slugIn = trim((string)($_POST['slug'] ?? ''));

    $containerWidth = (int)($_POST['containerWidth'] ?? 0);
    $accent = trim((string)($_POST['accent'] ?? ''));
    $logoFileId = (int)($_POST['logoFileId'] ?? 0);

    // валидация name/slug
    if ($name === '') {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'NAME_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $slug = $slugIn !== '' ? sb_slugify($slugIn) : sb_slugify($name);

    // проверка цвета (простая)
    if ($accent !== '' && !preg_match('~^#[0-9a-fA-F]{6}$~', $accent)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ACCENT_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    // ширина контейнера (если 0 — оставим как есть/дефолт)
    if ($containerWidth !== 0) {
        if ($containerWidth < 900) $containerWidth = 900;
        if ($containerWidth > 1600) $containerWidth = 1600;
    }

    // logoFileId опционально, но если задан — должен лежать в папке сайта
    if ($logoFileId > 0) {
        if (!sb_disk_file_belongs_to_site($siteId, $logoFileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$logoFileId], JSON_UNESCAPED_UNICODE);
            exit;
        }
    }

    $sites = sb_read_sites();
    $found = false;

    // уникальность slug среди сайтов
    $existing = array_map(fn($x) => (string)($x['slug'] ?? ''), array_filter($sites, fn($s) => (int)($s['id'] ?? 0) !== $siteId));
    $base = $slug; $i = 2;
    while (in_array($slug, $existing, true)) { $slug = $base.'-'.$i; $i++; }

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) !== $siteId) continue;

        $s['name'] = $name;
        $s['slug'] = $slug;

        // settings
        if (!isset($s['settings']) || !is_array($s['settings'])) $s['settings'] = [];

        if ($containerWidth !== 0) $s['settings']['containerWidth'] = $containerWidth;
        if ($accent !== '') $s['settings']['accent'] = $accent;

        // logo можно сбрасывать в 0
        $s['settings']['logoFileId'] = $logoFileId;

        $s['updatedAt'] = date('c');
        $s['updatedBy'] = (int)$USER->GetID();

        $found = true;
        break;
    }
    unset($s);

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_sites($sites);

    echo json_encode([
        'ok' => true,
        'siteId' => $siteId,
        'name' => $name,
        'slug' => $slug,
    ], JSON_UNESCAPED_UNICODE);
    exit;
}
```

### Маленькое улучшение (желательно): дефолт settings при `site.create`

В твоём `site.create` добавь в массив `$site`:

```php
'settings' => [
  'containerWidth' => 1100,
  'accent' => '#2563eb',
  'logoFileId' => 0,
],
```

Это не обязательно, но красиво.

---

## 3) Готовый `settings.php` (целиком)

Создай файл: **`/local/sitebuilder/settings.php`** и вставь:

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
    body { font-family: Arial, sans-serif; margin:0; background:#f6f7f8; color:#111; }
    .top { background:#fff; border-bottom:1px solid #e5e7ea; padding:12px 16px; display:flex; gap:10px; align-items:center; flex-wrap:wrap; }
    .content { padding: 18px; }
    .card { background:#fff; border:1px solid #e5e7ea; border-radius:14px; padding:16px; }
    .muted { color:#6a737f; }
    a { color:#0b57d0; text-decoration:none; }
    a:hover { text-decoration:underline; }
    code { background:#f3f4f6; padding:2px 6px; border-radius:6px; }

    .grid { display:grid; gap:12px; margin-top:12px; }
    @media (min-width: 820px){ .grid { grid-template-columns: 1fr 1fr; } }

    .field { margin-top:10px; }
    .field label { display:block; font-size:12px; color:#6a737f; margin-bottom:4px; }
    .input, select { width:100%; padding:8px; border:1px solid #d0d7de; border-radius:10px; box-sizing:border-box; }
    .row { display:flex; gap:10px; flex-wrap:wrap; align-items:center; justify-content:space-between; }
    .preview { margin-top:12px; border:1px dashed #e5e7ea; border-radius:14px; padding:12px; background:#fafafa; }
    .logoPrev { margin-top:10px; max-width:220px; border:1px solid #eee; border-radius:14px; overflow:hidden; background:#fff; }
    .logoPrev img { display:block; width:100%; height:auto; }
  </style>
</head>
<body>
  <div class="top">
    <a href="/local/sitebuilder/index.php">← Назад</a>
    <div class="muted">Настройки сайта</div>
    <div class="muted">|</div>
    <div><b>siteId:</b> <code><?= (int)$siteId ?></code></div>
    <div style="flex:1;"></div>
    <button class="ui-btn ui-btn-light" id="btnReload">Обновить</button>
    <button class="ui-btn ui-btn-primary" id="btnSave">Сохранить</button>
  </div>

  <div class="content">
    <div class="card">
      <div class="muted">Здесь настраиваем внешний вид и базовые параметры сайта. Требуются права <b>EDITOR+</b>.</div>

      <div class="grid">
        <div>
          <h3 style="margin:14px 0 0;">Основное</h3>

          <div class="field">
            <label>Название сайта</label>
            <input id="f_name" class="input" placeholder="Название">
          </div>

          <div class="field">
            <label>Slug</label>
            <input id="f_slug" class="input" placeholder="slug (если пусто — пересчитаем)">
          </div>

          <div class="field">
            <label>Верхнее меню</label>
            <select id="f_topMenu" class="input">
              <option value="0">— не выбрано —</option>
            </select>
            <div class="muted" style="margin-top:6px;font-size:12px;">Это то меню, которое показываем в view/public.</div>
          </div>
        </div>

        <div>
          <h3 style="margin:14px 0 0;">Визуал</h3>

          <div class="field">
            <label>Ширина контейнера (900..1600)</label>
            <input id="f_container" class="input" type="number" min="900" max="1600" step="10" placeholder="1100">
          </div>

          <div class="field">
            <label>Accent цвет (hex)</label>
            <input id="f_accent" class="input" placeholder="#2563eb">
          </div>

          <div class="field">
            <label>Логотип (файл из “Файлы” сайта)</label>
            <select id="f_logo" class="input">
              <option value="0">— без логотипа —</option>
            </select>
            <div id="logoPrev" class="logoPrev" style="display:none;">
              <img id="logoPrevImg" src="" alt="">
            </div>
          </div>

          <div class="preview" id="previewBox">
            <div style="font-weight:800;">Превью</div>
            <div class="muted" style="margin-top:6px;">Цвет и ширина применяются в будущем (в view/public), но мы уже проверяем визуально.</div>
            <div style="margin-top:10px;">
              <span style="display:inline-block;padding:8px 12px;border-radius:999px;border:1px solid #e5e7ea;background:#fff;">
                Пример пилюли меню
              </span>
              <span id="accentDot" style="display:inline-block;width:10px;height:10px;border-radius:999px;margin-left:8px;vertical-align:middle;"></span>
            </div>
          </div>
        </div>
      </div>

      <div style="margin-top:14px;" class="muted">
        Подсказка: если хочешь, позже мы применим эти настройки в <code>view.php</code> и <code>public.php</code>.
      </div>
    </div>
  </div>

<script>
BX.ready(function(){
  const siteId = <?= (int)$siteId ?>;

  const fName = document.getElementById('f_name');
  const fSlug = document.getElementById('f_slug');
  const fTopMenu = document.getElementById('f_topMenu');

  const fContainer = document.getElementById('f_container');
  const fAccent = document.getElementById('f_accent');
  const fLogo = document.getElementById('f_logo');

  const btnReload = document.getElementById('btnReload');
  const btnSave = document.getElementById('btnSave');

  const logoPrev = document.getElementById('logoPrev');
  const logoPrevImg = document.getElementById('logoPrevImg');
  const accentDot = document.getElementById('accentDot');

  function notify(msg){
    BX.UI.Notification.Center.notify({ content: msg });
  }

  function api(action, data){
    return new Promise((resolve, reject) => {
      BX.ajax({
        url: '/local/sitebuilder/api.php',
        method: 'POST',
        dataType: 'json',
        data: Object.assign({ action, sessid: BX.bitrix_sessid() }, data || {}),
        onsuccess: resolve,
        onfailure: reject
      });
    });
  }

  function downloadUrl(fileId){
    return `/local/sitebuilder/download.php?siteId=${siteId}&fileId=${fileId}`;
  }

  function applyPreview(){
    const hex = (fAccent.value || '').trim();
    accentDot.style.background = /^#[0-9a-fA-F]{6}$/.test(hex) ? hex : '#e5e7ea';
  }

  async function load(){
    if (!siteId) { notify('siteId не задан'); return; }

    let siteRes, menusRes, filesRes;
    try {
      [siteRes, menusRes, filesRes] = await Promise.all([
        api('site.get', { siteId }),
        api('menu.list', { siteId }),
        api('file.list', { siteId })
      ]);
    } catch(e){
      notify('Ошибка загрузки данных');
      return;
    }

    if (!siteRes || siteRes.ok !== true) { notify('Не удалось получить site.get'); return; }
    const site = siteRes.site || {};

    const settings = (site.settings && typeof site.settings === 'object') ? site.settings : {};

    fName.value = site.name || '';
    fSlug.value = site.slug || '';

    fContainer.value = settings.containerWidth || 1100;
    fAccent.value = settings.accent || '#2563eb';

    const topMenuId = parseInt(site.topMenuId || 0, 10) || 0;

    // menus
    const menus = (menusRes && menusRes.ok===true) ? (menusRes.menus||[]) : [];
    fTopMenu.innerHTML = `<option value="0">— не выбрано —</option>` + menus.map(m => {
      const id = parseInt(m.id||0,10);
      const sel = id === topMenuId ? 'selected' : '';
      return `<option value="${id}" ${sel}>${BX.util.htmlspecialchars(m.name || ('Меню #' + id))} (id ${id})</option>`;
    }).join('');

    // files for logo
    const files = (filesRes && filesRes.ok===true) ? (filesRes.files||[]) : [];
    const curLogoId = parseInt(settings.logoFileId || 0, 10) || 0;

    fLogo.innerHTML = `<option value="0">— без логотипа —</option>` + files.map(f => {
      const id = parseInt(f.id||0,10);
      const sel = id === curLogoId ? 'selected' : '';
      return `<option value="${id}" ${sel}>${BX.util.htmlspecialchars(f.name)} (id ${id})</option>`;
    }).join('');

    const setLogoPrev = () => {
      const id = parseInt(fLogo.value||'0',10)||0;
      if (!id) { logoPrev.style.display='none'; logoPrevImg.src=''; return; }
      logoPrev.style.display='block';
      logoPrevImg.src = downloadUrl(id);
    };
    setLogoPrev();

    applyPreview();

    fLogo.onchange = setLogoPrev;
    fAccent.oninput = applyPreview;
  }

  async function save(){
    const name = (fName.value || '').trim();
    const slug = (fSlug.value || '').trim();
    const topMenuId = parseInt(fTopMenu.value || '0', 10) || 0;

    const containerWidth = parseInt(fContainer.value || '0', 10) || 0;
    const accent = (fAccent.value || '').trim();
    const logoFileId = parseInt(fLogo.value || '0', 10) || 0;

    if (!name){ notify('Название не может быть пустым'); return; }

    // 1) обновляем базу сайта + settings
    let upd;
    try{
      upd = await api('site.update', { siteId, name, slug, containerWidth, accent, logoFileId });
    }catch(e){
      notify('Ошибка site.update');
      return;
    }
    if (!upd || upd.ok !== true){
      notify('Не удалось сохранить (возможно нет прав)');
      return;
    }

    // 2) topMenuId — если у тебя уже есть отдельный menu.setTop — можно не трогать.
    // Но мы сделаем напрямую через site.update, если ты это реализовал в нём позже.
    // Сейчас topMenuId сохраняем отдельным action’ом, если он у тебя уже есть.
    // Если нет — просто оставим на будущее.
    // (Если хочешь — мы добавим в site.update поддержку topMenuId следующим шагом.)
    if (topMenuId > 0) {
      // у тебя уже есть menu.setTop -> он пишет в site.topMenuId.
      try{
        const r = await api('menu.setTop', { siteId, menuId: topMenuId });
        if (r && r.ok === true) {
          notify('Сохранено');
          await load();
          return;
        }
      }catch(e){
        // не критично
      }
    }

    notify('Сохранено');
    await load();
  }

  btnReload.onclick = load;
  btnSave.onclick = save;

  load();
});
</script>

</body>
</html>
```

> Важно: `settings.php` использует **уже существующий** `menu.setTop`, который у тебя работает.
> Если захочешь, мы позже расширим `site.update`, чтобы он тоже умел `topMenuId` напрямую (без второго вызова).

---

## Что дальше после этого шага

Ты сказал: “после этого переходим к settings.php + api.site.update” — мы это сделали.

Следующий логичный пункт (и он прям про визуал/возможности):

1. Применить `site.settings` в `view.php` (containerWidth + accent + logo)
2. Потом то же в `public.php` (публичный рендер)

Говори “ок”, и я дам точечные изменения в `view.php` (пару мест: CSS root + логотип в шапке), чтобы настройки реально влияли на внешний вид.
