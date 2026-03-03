Ок, следующий шаг — **settings.php + api.site.update** (настройки сайта: ширина контейнера, акцент, логотип из файлов сайта). Это логично, потому что ты уже используешь `site.settings` в `view.php`.

Ниже даю сразу готовый план и код.

---

## 1) Добавляем action `site.update` в `api.php`

Вставь **перед самым концом** (до `http_response_code(400) ... UNKNOWN_ACTION`).

```php
if ($action === 'site.update') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    // Настройки меняют только EDITOR+
    sb_require_editor($siteId);

    $sites = sb_read_sites();
    $found = false;

    $name = trim((string)($_POST['name'] ?? ''));
    $slugIn = trim((string)($_POST['slug'] ?? ''));

    // settings
    $containerWidth = (int)($_POST['containerWidth'] ?? 0);
    $accent = trim((string)($_POST['accent'] ?? ''));
    $logoFileId = (int)($_POST['logoFileId'] ?? 0);

    // Нормализация
    if ($containerWidth !== 0) {
        if ($containerWidth < 900) $containerWidth = 900;
        if ($containerWidth > 1600) $containerWidth = 1600;
    }

    if ($accent !== '' && !preg_match('~^#[0-9a-fA-F]{6}$~', $accent)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ACCENT_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    // Лого: опционально, но если задано — файл должен быть в папке сайта
    if ($logoFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $logoFileId)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$logoFileId], JSON_UNESCAPED_UNICODE);
        exit;
    }

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) !== $siteId) continue;

        // name/slug (опционально)
        if ($name !== '') $s['name'] = $name;

        if ($slugIn !== '') {
            $newSlug = sb_slugify($slugIn);

            // уникальность slug среди сайтов
            $existing = array_map(fn($x) => (string)($x['slug'] ?? ''), array_filter($sites, fn($x)=> (int)($x['id'] ?? 0) !== $siteId));
            $base = $newSlug; $i = 2;
            while (in_array($newSlug, $existing, true)) { $newSlug = $base.'-'.$i; $i++; }

            $s['slug'] = $newSlug;
        }

        // settings merge
        $set = (isset($s['settings']) && is_array($s['settings'])) ? $s['settings'] : [];

        if ($containerWidth !== 0) $set['containerWidth'] = $containerWidth;
        if ($accent !== '') $set['accent'] = $accent;
        // logoFileId: разрешаем 0 чтобы "убрать логотип"
        if (isset($_POST['logoFileId'])) $set['logoFileId'] = $logoFileId;

        $s['settings'] = $set;

        $s['updatedAt'] = date('c');
        $s['updatedBy'] = (int)$USER->GetID();

        $found = true;
        break;
    }
    unset($s);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_sites($sites);

    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

## 2) Делаем `settings.php` (страница настроек сайта)

Создай файл:
`/local/sitebuilder/settings.php`

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
    .card { max-width: 980px; margin:0 auto; background:#fff; border:1px solid #e5e7ea; border-radius:14px; padding:16px; }
    .muted { color:#6a737f; }
    code { background:#f3f4f6; padding:2px 6px; border-radius:6px; }
    .grid { display:grid; gap:12px; margin-top:12px; }
    @media (min-width: 860px){ .grid { grid-template-columns: 1fr 1fr; } }
    .field label { display:block; font-size:12px; color:#6a737f; margin-bottom:6px; }
    .input { width:100%; padding:9px; border:1px solid #d0d7de; border-radius:10px; box-sizing:border-box; }
    .row { display:flex; gap:10px; flex-wrap:wrap; align-items:center; }
    .preview { margin-top:12px; border:1px dashed #e5e7ea; border-radius:14px; padding:12px; }
    .brand { display:flex; gap:10px; align-items:center; }
    .brandMark { width:36px; height:36px; border-radius:12px; display:flex; align-items:center; justify-content:center; background:#eef2ff; border:1px solid #c7d2fe; font-weight:800; }
    .logoPrev img { max-height:36px; width:auto; display:block; border-radius:12px; border:1px solid #eee; background:#fff; }
    .actions { margin-top:14px; display:flex; gap:10px; flex-wrap:wrap; }
  </style>
</head>
<body>
  <div class="top">
    <a href="/local/sitebuilder/index.php">← Назад</a>
    <div class="muted">Настройки сайта</div>
    <div class="muted">|</div>
    <div><b>siteId:</b> <code><?= (int)$siteId ?></code></div>
    <div style="flex:1;"></div>
    <a class="ui-btn ui-btn-light" target="_blank" href="/local/sitebuilder/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
    <a class="ui-btn ui-btn-light" target="_blank" href="/local/sitebuilder/files.php?siteId=<?= (int)$siteId ?>">Файлы</a>
  </div>

  <div class="content">
    <div class="card">
      <h2 style="margin:0;">Настройки оформления</h2>
      <div class="muted" style="margin-top:8px;">
        Здесь задаём внешний вид для <code>view.php</code>: ширина контейнера, цвет, логотип (из файлов сайта).
      </div>

      <div class="grid">
        <div class="field">
          <label>Название сайта</label>
          <input class="input" id="sb_name" placeholder="Например: Лаборатория">
        </div>

        <div class="field">
          <label>Slug (необязательно)</label>
          <input class="input" id="sb_slug" placeholder="например: lab">
        </div>

        <div class="field">
          <label>Ширина контейнера (900..1600)</label>
          <input class="input" id="sb_width" type="number" min="900" max="1600" step="10" value="1100">
        </div>

        <div class="field">
          <label>Accent (HEX, например #2563eb)</label>
          <input class="input" id="sb_accent" value="#2563eb">
        </div>

        <div class="field" style="grid-column: 1 / -1;">
          <label>Логотип (файл из “Файлы” сайта)</label>
          <div class="row">
            <select class="input" id="sb_logo" style="max-width:520px;">
              <option value="0">— без логотипа —</option>
            </select>
            <button class="ui-btn ui-btn-light" id="btnReloadFiles" type="button">Обновить файлы</button>
          </div>
          <div class="muted" style="margin-top:8px;">Для логотипа лучше PNG/SVG, высота ~32-40px.</div>
        </div>
      </div>

      <div class="preview" id="prevBox">
        <div class="muted" style="font-size:12px;">Превью шапки (пример)</div>
        <div class="row" style="justify-content:space-between; margin-top:10px;">
          <div class="brand">
            <div class="logoPrev" id="logoPrev" style="display:none;"></div>
            <div class="brandMark" id="brandMark">SB</div>
            <div>
              <div style="font-weight:800;" id="prevName">Site</div>
              <div class="muted" style="font-size:12px;">Page</div>
            </div>
          </div>
          <div class="row">
            <span class="ui-btn ui-btn-light">Меню</span>
            <span class="ui-btn ui-btn-primary">Редактор</span>
          </div>
        </div>
      </div>

      <div class="actions">
        <button class="ui-btn ui-btn-primary" id="btnSave">Сохранить</button>
        <a class="ui-btn ui-btn-light" target="_blank" id="btnOpenAnyPage" href="javascript:void(0)">Открыть любую страницу</a>
      </div>

      <div class="muted" style="margin-top:10px;">
        Нужно право <b>EDITOR+</b> на сайт. Ошибка “FORBIDDEN” — значит роль ниже.
      </div>
    </div>
  </div>

<script>
BX.ready(function () {
  const siteId = <?= (int)$siteId ?>;

  const elName = document.getElementById('sb_name');
  const elSlug = document.getElementById('sb_slug');
  const elWidth = document.getElementById('sb_width');
  const elAccent = document.getElementById('sb_accent');
  const elLogo = document.getElementById('sb_logo');
  const btnSave = document.getElementById('btnSave');
  const btnReloadFiles = document.getElementById('btnReloadFiles');

  const prevBox = document.getElementById('prevBox');
  const prevName = document.getElementById('prevName');
  const brandMark = document.getElementById('brandMark');
  const logoPrev = document.getElementById('logoPrev');
  const btnOpenAnyPage = document.getElementById('btnOpenAnyPage');

  function notify(msg){ BX.UI.Notification.Center.notify({ content: msg }); }

  function api(action, data) {
    return new Promise((resolve, reject) => {
      BX.ajax({
        url: '/local/sitebuilder/api.php',
        method: 'POST',
        dataType: 'json',
        data: Object.assign({ action, sessid: BX.bitrix_sessid(), siteId }, data || {}),
        onsuccess: resolve,
        onfailure: reject
      });
    });
  }

  function fileDownloadUrl(fileId){
    return `/local/sitebuilder/download.php?siteId=${siteId}&fileId=${fileId}`;
  }

  function applyPreview(){
    const accent = (elAccent.value || '#2563eb').trim();
    prevBox.style.borderColor = '#e5e7ea';
    brandMark.style.color = accent;
    brandMark.style.borderColor = '#c7d2fe';
    brandMark.style.background = '#eef2ff';
    prevName.textContent = elName.value || 'Site';
  }

  async function loadSite(){
    const r = await api('site.get', { siteId });
    if (!r || r.ok !== true) { notify('Не удалось загрузить site.get'); return; }

    const s = r.site || {};
    elName.value = s.name || '';
    elSlug.value = s.slug || '';

    const set = (s.settings && typeof s.settings === 'object') ? s.settings : {};
    elWidth.value = parseInt(set.containerWidth || 1100, 10);
    elAccent.value = set.accent || '#2563eb';
    elLogo.value = parseInt(set.logoFileId || 0, 10);

    applyPreview();
  }

  async function loadFiles(){
    const r = await api('file.list', { siteId });
    if (!r || r.ok !== true) { notify('Не удалось загрузить file.list'); return; }

    const files = r.files || [];
    const selected = parseInt(elLogo.value || '0', 10) || 0;

    const opts = ['<option value="0">— без логотипа —</option>']
      .concat(files.map(f => {
        const s = (parseInt(f.id,10) === selected) ? 'selected' : '';
        return `<option value="${f.id}" ${s}>${BX.util.htmlspecialchars(f.name)} (id ${f.id})</option>`;
      }));

    elLogo.innerHTML = opts.join('');

    const fid = parseInt(elLogo.value || '0', 10) || 0;
    if (fid > 0){
      logoPrev.style.display = 'block';
      logoPrev.innerHTML = `<img src="${fileDownloadUrl(fid)}" alt="logo">`;
    } else {
      logoPrev.style.display = 'none';
      logoPrev.innerHTML = '';
    }
  }

  async function loadAnyPageLink(){
    try {
      const r = await api('page.list', { siteId });
      if (!r || r.ok !== true) return;
      const pages = r.pages || [];
      if (!pages.length) return;
      const pid = pages[0].id;
      btnOpenAnyPage.href = `/local/sitebuilder/view.php?siteId=${siteId}&pageId=${pid}`;
    } catch(e){}
  }

  btnReloadFiles.onclick = loadFiles;

  elAccent.addEventListener('input', applyPreview);
  elName.addEventListener('input', applyPreview);
  elLogo.addEventListener('change', () => {
    const fid = parseInt(elLogo.value || '0', 10) || 0;
    if (fid > 0){
      logoPrev.style.display = 'block';
      logoPrev.innerHTML = `<img src="${fileDownloadUrl(fid)}" alt="logo">`;
    } else {
      logoPrev.style.display = 'none';
      logoPrev.innerHTML = '';
    }
  });

  btnSave.onclick = async () => {
    const name = (elName.value || '').trim();
    const slug = (elSlug.value || '').trim();
    const containerWidth = parseInt(elWidth.value || '1100', 10);
    const accent = (elAccent.value || '').trim();
    const logoFileId = parseInt(elLogo.value || '0', 10) || 0;

    if (!name){ notify('Название сайта не может быть пустым'); return; }
    if (accent && !/^#[0-9a-fA-F]{6}$/.test(accent)){ notify('Accent должен быть в формате #RRGGBB'); return; }
    if (containerWidth < 900 || containerWidth > 1600){ notify('Ширина должна быть 900..1600'); return; }

    const r = await api('site.update', { siteId, name, slug, containerWidth, accent, logoFileId });
    if (!r || r.ok !== true) { notify('Не удалось сохранить (site.update)'); return; }

    notify('Сохранено');
  };

  // init
  (async () => {
    await loadSite();
    await loadFiles();
    await loadAnyPageLink();
    applyPreview();
  })();
});
</script>
</body>
</html>
```

---

## 3) Куда вставить “3 пункт” в `view.php` (ты спрашивал)

То, что у тебя в `view.php` начинается с:

```php
$settings = (isset($site['settings']) && is_array($site['settings'])) ? $site['settings'] : [];
...
$containerWidth ...
$accent ...
$logoFileId ...
```

— это и есть тот самый “3 пункт”. Его нужно держать **после**:

* того как нашли `$site` и `$page`
* и **после** проверки `if (!$site || !$page) { ... exit; }`

То есть у тебя сейчас всё стоит **правильно**: блок настроек идёт сразу после `exit;` и до расчётов меню/страниц/блоков.

---

## 4) Что дальше после этого

1. Добавь кнопку **“Настройки”** в `index.php` рядом с “Меню/Файлы/Доступы”:

   * ссылка: `/local/sitebuilder/settings.php?siteId=${s.id}`

2. Проверь сценарий:

   * загрузил логотип через “Файлы”
   * выбрал в settings.php логотип + цвет
   * открыл любую страницу — в `view.php` подтянулось

Если хочешь — пришли свой `index.php`, я вставлю кнопку “Настройки” ровно в твой текущий дизайн, без поломок.
