Ок, пропускаем форму. Тогда следующий “ядровой” шаг для тильда-аналога — **публикация сайта**: чтобы у сайта был **публичный URL**, который открывается **без editor**, и можно выбрать “какая страница главная”.

Сделаем это без красивых ссылок, максимально просто и надёжно:

## Что добавляем

1. В `pages.json`/записи страницы — флаг `isHome` (или `homePageId` в site).
2. Новый публичный файл: **`/local/sitebuilder/public.php`**

   * принимает `siteId` и `slug` страницы (или пусто → откроет home)
   * грузит блоки и рендерит как `view.php`, но без служебной шапки и без требований прав (или с простой проверкой “публичен” — добавим позже)
3. В `index.php` (или где список страниц) — кнопка “Сделать главной”.

Я сейчас дам **самый быстрый вариант**: хранить **homePageId внутри site** (это проще и быстрее, чем флаги на страницах).

---

# 1) api.php — добавить `site.setHome`

Добавь action (рядом с другими `site.*`):

```php
if ($action === 'site.setHome') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $pageId = (int)($_POST['pageId'] ?? 0);

    if ($siteId <= 0 || $pageId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_PAGE_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    // только editor+ (или owner, решишь сам)
    sb_require_editor($siteId);

    $page = sb_find_page($pageId);
    if (!$page || (int)($page['siteId'] ?? 0) !== $siteId) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_IN_SITE'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $sites = sb_read_sites();
    $found = false;

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $s['homePageId'] = $pageId;
            $s['updatedAt'] = date('c');
            $s['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($s);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_sites($sites);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

# 2) public.php — новый файл (публичный рендер)

Создай файл: **`/local/sitebuilder/public.php`**

```php
<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

header('Content-Type: text/html; charset=UTF-8');

function h($s): string { return htmlspecialcharsbx((string)$s); }

function sb_data_path(string $file): string {
    return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/' . $file;
}
function sb_read_json_file(string $file): array {
    $path = sb_data_path($file);
    if (!file_exists($path)) return [];
    $raw = file_get_contents($path);
    $data = json_decode((string)$raw, true);
    return is_array($data) ? $data : [];
}
function sb_read_sites(): array { return sb_read_json_file('sites.json'); }
function sb_read_pages(): array { return sb_read_json_file('pages.json'); }
function sb_read_blocks(): array { return sb_read_json_file('blocks.json'); }

function find_site(int $siteId): ?array {
    foreach (sb_read_sites() as $s) if ((int)($s['id'] ?? 0) === $siteId) return $s;
    return null;
}
function find_page(int $pageId): ?array {
    foreach (sb_read_pages() as $p) if ((int)($p['id'] ?? 0) === $pageId) return $p;
    return null;
}
function find_page_by_slug(int $siteId, string $slug): ?array {
    foreach (sb_read_pages() as $p) {
        if ((int)($p['siteId'] ?? 0) === $siteId && (string)($p['slug'] ?? '') === $slug) return $p;
    }
    return null;
}

$siteId = (int)($_GET['siteId'] ?? 0);
$slug = trim((string)($_GET['p'] ?? '')); // p=page-slug (опционально)

if ($siteId <= 0) {
    http_response_code(404);
    echo 'Site not found';
    exit;
}

$site = find_site($siteId);
if (!$site) { http_response_code(404); echo 'Site not found'; exit; }

// выбираем страницу
$page = null;
if ($slug !== '') $page = find_page_by_slug($siteId, $slug);

if (!$page) {
    $homeId = (int)($site['homePageId'] ?? 0);
    if ($homeId > 0) $page = find_page($homeId);
}
if (!$page) {
    // если home не задан — берём первую страницу
    foreach (sb_read_pages() as $p) {
        if ((int)($p['siteId'] ?? 0) === $siteId) { $page = $p; break; }
    }
}
if (!$page) { http_response_code(404); echo 'Page not found'; exit; }

$pageId = (int)($page['id'] ?? 0);

// блоки страницы
$blocks = array_values(array_filter(sb_read_blocks(), fn($b) => (int)($b['pageId'] ?? 0) === $pageId));
usort($blocks, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

function file_url(int $siteId, int $fileId): string {
    return '/local/sitebuilder/download.php?siteId=' . $siteId . '&fileId=' . $fileId;
}

?><!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title><?=h($site['name'] ?? 'Site')?> — <?=h($page['title'] ?? '')?></title>
  <style>
    body{font-family:Arial,sans-serif;margin:0;background:#fff;color:#111;}
    .wrap{max-width:980px;margin:0 auto;padding:24px;}
    .muted{color:#6a737f;}
    .btn{display:inline-block;padding:10px 14px;border-radius:12px;border:1px solid #e5e7ea;text-decoration:none;}
    .btn.primary{background:#2563eb;color:#fff;border-color:#2563eb;}
    .btn.secondary{background:#fff;color:#111;}
    img{max-width:100%;height:auto;border-radius:14px;border:1px solid #eee;background:#fafafa;}
    .block{margin-top:14px;}
    .cols2{display:grid;gap:14px;}
    @media (min-width: 820px){ .cols2{grid-template-columns:1fr 1fr;} }
    .cardItem{background:#fff;border:1px solid #eee;border-radius:16px;padding:14px;}
    .cardsGrid{display:grid;gap:14px;}
    @media (max-width: 768px){ .cardsGrid{grid-template-columns:1fr !important;} }
  </style>
</head>
<body>
<div class="wrap">
  <?php foreach ($blocks as $b): ?>
    <?php $type = (string)($b['type'] ?? ''); $c = $b['content'] ?? []; ?>
    <div class="block">
      <?php if ($type === 'text'): ?>
        <div style="white-space:pre-wrap; line-height:1.7;"><?=h($c['text'] ?? '')?></div>
      <?php elseif ($type === 'heading'): ?>
        <?php
          $lvl = in_array(($c['level'] ?? 'h2'), ['h1','h2','h3'], true) ? $c['level'] : 'h2';
          $al = in_array(($c['align'] ?? 'left'), ['left','center','right'], true) ? $c['align'] : 'left';
        ?>
        <<?=h($lvl)?> style="margin:0; text-align:<?=h($al)?>;"><?=h($c['text'] ?? '')?></<?=h($lvl)?>>
      <?php elseif ($type === 'image'): ?>
        <?php $fid = (int)($c['fileId'] ?? 0); if ($fid>0): ?>
          <img src="<?=h(file_url($siteId, $fid))?>" alt="<?=h($c['alt'] ?? '')?>">
        <?php endif; ?>
      <?php elseif ($type === 'button'): ?>
        <?php
          $url = (string)($c['url'] ?? '');
          $txt = (string)($c['text'] ?? 'Open');
          $v = (string)($c['variant'] ?? 'primary');
          $cls = ($v === 'secondary') ? 'secondary' : 'primary';
        ?>
        <a class="btn <?=h($cls)?>" href="<?=h($url)?>" <?=preg_match('~^https?://~i',$url)?'target="_blank" rel="noopener noreferrer"':''?>><?=h($txt)?></a>
      <?php elseif ($type === 'columns2'): ?>
        <?php
          $ratio = (string)($c['ratio'] ?? '50-50');
          $tpl = '1fr 1fr';
          if ($ratio === '33-67') $tpl = '1fr 2fr';
          if ($ratio === '67-33') $tpl = '2fr 1fr';
        ?>
        <div class="cols2" style="grid-template-columns:<?=h($tpl)?>;">
          <div style="white-space:pre-wrap;line-height:1.7;"><?=h($c['left'] ?? '')?></div>
          <div style="white-space:pre-wrap;line-height:1.7;"><?=h($c['right'] ?? '')?></div>
        </div>
      <?php elseif ($type === 'gallery'): ?>
        <?php
          $cols = (int)($c['columns'] ?? 3); if (!in_array($cols,[2,3,4],true)) $cols=3;
          $tpl = ($cols===2)?'1fr 1fr':(($cols===4)?'1fr 1fr 1fr 1fr':'1fr 1fr 1fr');
          $imgs = is_array($c['images'] ?? null) ? $c['images'] : [];
        ?>
        <div style="display:grid;gap:14px;grid-template-columns:<?=h($tpl)?>;">
          <?php foreach ($imgs as $it): $fid=(int)($it['fileId']??0); if($fid<=0) continue; ?>
            <img src="<?=h(file_url($siteId,$fid))?>" alt="<?=h($it['alt']??'')?>">
          <?php endforeach; ?>
        </div>
      <?php elseif ($type === 'spacer'): ?>
        <?php $hgt=(int)($c['height']??40); $line = !empty($c['line']); ?>
        <div style="height:<?=h($hgt)?>px; position:relative;">
          <?php if ($line): ?><div style="position:absolute;left:0;right:0;top:50%;height:1px;background:#e5e7ea;"></div><?php endif; ?>
        </div>
      <?php elseif ($type === 'card'): ?>
        <div class="cardItem">
          <div style="font-weight:700;"><?=h($c['title']??'')?></div>
          <div class="muted" style="margin-top:8px;white-space:pre-wrap;line-height:1.7;"><?=h($c['text']??'')?></div>
          <?php $fid=(int)($c['imageFileId']??0); if($fid>0): ?><div style="margin-top:10px;"><img src="<?=h(file_url($siteId,$fid))?>" alt=""></div><?php endif; ?>
          <?php $url=(string)($c['buttonUrl']??''); if($url!==''): ?>
            <a class="btn secondary" style="margin-top:10px;" href="<?=h($url)?>" <?=preg_match('~^https?://~i',$url)?'target="_blank" rel="noopener noreferrer"':''?>><?=h(($c['buttonText']??'Открыть'))?></a>
          <?php endif; ?>
        </div>
      <?php elseif ($type === 'cards'): ?>
        <?php
          $cols = (int)($c['columns'] ?? 3); if (!in_array($cols,[2,3,4],true)) $cols=3;
          $tpl = ($cols===2)?'1fr 1fr':(($cols===4)?'1fr 1fr 1fr 1fr':'1fr 1fr 1fr');
          $items = is_array($c['items'] ?? null) ? $c['items'] : [];
        ?>
        <div class="cardsGrid" style="grid-template-columns:<?=h($tpl)?>;">
          <?php foreach ($items as $it): if (!is_array($it)) continue; $t=trim((string)($it['title']??'')); if($t==='') continue; ?>
            <div class="cardItem">
              <div style="font-weight:700;"><?=h($t)?></div>
              <?php $tx=(string)($it['text']??''); if($tx!==''): ?><div class="muted" style="margin-top:8px;white-space:pre-wrap;line-height:1.7;"><?=h($tx)?></div><?php endif; ?>
              <?php $fid=(int)($it['imageFileId']??0); if($fid>0): ?><div style="margin-top:10px;"><img src="<?=h(file_url($siteId,$fid))?>" alt=""></div><?php endif; ?>
              <?php $url=(string)($it['buttonUrl']??''); if($url!==''): ?>
                <a class="btn secondary" style="margin-top:10px;" href="<?=h($url)?>" <?=preg_match('~^https?://~i',$url)?'target="_blank" rel="noopener noreferrer"':''?>><?=h(($it['buttonText']??'Открыть'))?></a>
              <?php endif; ?>
            </div>
          <?php endforeach; ?>
        </div>
      <?php endif; ?>
    </div>
  <?php endforeach; ?>
</div>
</body>
</html>
```

✅ Теперь у тебя есть “публичный сайт” по ссылке:
`/local/sitebuilder/public.php?siteId=1`
или конкретная страница:
`/local/sitebuilder/public.php?siteId=1&p=about`

---

# 3) Что делаем дальше после этого

После того как `public.php` откроется:

* добавим в `index.php` кнопку **“Открыть публично”**
* добавим “Сделать главной” (homePageId)
* и (если захочешь) сделаем красивый URL вида `/sb/site-slug/page-slug` через rewrite — но это потом.

Сейчас скажи: **где у тебя удобнее выбирать “главную страницу” — в списке страниц (index.php) или прямо в editor.php?**
Если не хочешь выбирать — сделаю по умолчанию “первая созданная”.
