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
function sb_read_menus(): array { return sb_read_json_file('menus.json'); }

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

function sb_menu_get_site_record(array $all, int $siteId): ?array {
    foreach ($all as $r) if ((int)($r['siteId'] ?? 0) === $siteId) return $r;
    return null;
}
function sb_menu_pick_main(array $siteRecord): ?array {
    $menus = $siteRecord['menus'] ?? [];
    if (!is_array($menus) || !count($menus)) return null;
    // пока берём первое меню как "главное"
    return $menus[0];
}

function public_page_url(int $siteId, ?array $page): string {
    $slug = $page ? (string)($page['slug'] ?? '') : '';
    if ($slug !== '') return '/local/sitebuilder/public.php?siteId=' . $siteId . '&p=' . urlencode($slug);
    return '/local/sitebuilder/public.php?siteId=' . $siteId;
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

$menuItems = [];
$allMenus = sb_read_menus();
$siteRec = sb_menu_get_site_record($allMenus, $siteId);
$mainMenu = $siteRec ? sb_menu_pick_main($siteRec) : null;

if ($mainMenu && is_array($mainMenu['items'] ?? null)) {
    $items = $mainMenu['items'];
    usort($items, fn($a,$b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));

    foreach ($items as $it) {
        if (!is_array($it)) continue;
        $type = (string)($it['type'] ?? '');
        $title = trim((string)($it['title'] ?? ''));
        if ($title === '') $title = 'Link';

        $href = '#';
        if ($type === 'page') {
            $pid = (int)($it['pageId'] ?? 0);
            $p = $pid > 0 ? find_page($pid) : null;
            // безопасность: только страницы этого сайта
            if ($p && (int)($p['siteId'] ?? 0) === $siteId) {
                $href = public_page_url($siteId, $p);
                if ($title === 'Link') $title = (string)($p['title'] ?? 'Page');
            } else {
                continue;
            }
        } elseif ($type === 'url') {
            $url = trim((string)($it['url'] ?? ''));
            if ($url === '') continue;
            $href = $url;
        } else {
            continue;
        }

        $menuItems[] = ['title' => $title, 'href' => $href];
    }
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

    .header{position:sticky;top:0;background:#fff;border-bottom:1px solid #eee;z-index:10;}
    .header .in{max-width:980px;margin:0 auto;padding:12px 24px;display:flex;gap:14px;align-items:center;flex-wrap:wrap;}
    .brand{font-weight:800;}
    .nav{display:flex;gap:12px;flex-wrap:wrap;}
    .nav a{color:#111;text-decoration:none;padding:6px 10px;border-radius:10px;}
    .nav a:hover{background:#f3f4f6;}
  </style>
</head>
<body>
<div class="header">
  <div class="in">
    <div class="brand"><?=h($site['name'] ?? 'Site')?></div>
    <?php if (count($menuItems)): ?>
      <nav class="nav">
        <?php foreach ($menuItems as $mi): ?>
          <a href="<?=h($mi['href'])?>" <?=preg_match('~^https?://~i',$mi['href'])?'target="_blank" rel="noopener noreferrer"':''?>>
            <?=h($mi['title'])?>
          </a>
        <?php endforeach; ?>
      </nav>
    <?php endif; ?>
  </div>
</div>
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
