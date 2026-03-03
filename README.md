<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

global $USER;

if (!$USER->IsAuthorized()) {
    LocalRedirect('/auth/');
}

header('Content-Type: text/html; charset=UTF-8');

$siteId = (int)($_GET['siteId'] ?? 0);
$pageId = (int)($_GET['pageId'] ?? 0);

function sb_data_path(string $file): string {
    return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/' . $file;
}
function sb_read_json(string $file): array {
    $path = sb_data_path($file);
    if (!file_exists($path)) return [];
    $raw = file_get_contents($path);
    $data = json_decode((string)$raw, true);
    return is_array($data) ? $data : [];
}
function h($s): string { return htmlspecialcharsbx((string)$s); }

function downloadUrl(int $siteId, int $fileId): string {
    return '/local/sitebuilder/download.php?siteId=' . $siteId . '&fileId=' . $fileId;
}

function viewUrl(int $siteId, int $pageId): string {
    return '/local/sitebuilder/view.php?siteId=' . $siteId . '&pageId=' . $pageId;
}

function sb_index_pages_by_id(array $pages, int $siteId): array {
    $byId = [];
    foreach ($pages as $p) {
        if ((int)($p['siteId'] ?? 0) !== $siteId) continue;
        $id = (int)($p['id'] ?? 0);
        if ($id > 0) $byId[$id] = $p;
    }
    return $byId;
}

function sb_build_breadcrumbs(array $pagesById, int $currentPageId): array {
    // возвращает массив страниц от корня к текущей (включая текущую)
    $chain = [];
    $guard = 0;
    $curId = $currentPageId;

    while ($curId > 0 && isset($pagesById[$curId])) {
        $chain[] = $pagesById[$curId];
        $parentId = (int)($pagesById[$curId]['parentId'] ?? 0);
        if ($parentId <= 0) break;
        $curId = $parentId;

        $guard++;
        if ($guard > 50) break; // защита от циклов
    }

    return array_reverse($chain);
}

function sb_get_children_pages(array $pages, int $siteId, int $parentId): array {
    $kids = array_values(array_filter($pages, function($p) use ($siteId, $parentId){
        return (int)($p['siteId'] ?? 0) === $siteId && (int)($p['parentId'] ?? 0) === $parentId;
    }));
    usort($kids, function($a, $b){
        $sa = (int)($a['sort'] ?? 500);
        $sb = (int)($b['sort'] ?? 500);
        if ($sa === $sb) return ((int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0));
        return $sa <=> $sb;
    });
    return $kids;
}

// load data
$sites = sb_read_json('sites.json');
$pages = sb_read_json('pages.json');
$blocksAll = sb_read_json('blocks.json');
$menusAll = sb_read_json('menus.json');

// find site/page
$site = null;
foreach ($sites as $s) {
    if ((int)($s['id'] ?? 0) === $siteId) { $site = $s; break; }
}

$topMenuId = (int)($site['topMenuId'] ?? 0);

$page = null;
foreach ($pages as $p) {
    if ((int)($p['id'] ?? 0) === $pageId && (int)($p['siteId'] ?? 0) === $siteId) { $page = $p; break; }
}

if (!$site || !$page) {
    http_response_code(404);
    ?>
    <!doctype html><html lang="ru"><head><meta charset="utf-8"><title>Не найдено</title></head>
    <body style="font-family:Arial;padding:24px;background:#f6f7f8;">
      <div style="background:#fff;border:1px solid #e5e7ea;border-radius:12px;padding:16px;">
        <h2 style="margin-top:0;">Страница не найдена</h2>
        <div>siteId=<?= (int)$siteId ?>, pageId=<?= (int)$pageId ?></div>
        <div style="margin-top:12px;"><a href="/local/sitebuilder/index.php">← Назад</a></div>
      </div>
    </body></html>
    <?php
    exit;
}

// site pages (for fallback nav / editor link)
$sitePages = array_values(array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId));
usort($sitePages, fn($a, $b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

$pagesById = sb_index_pages_by_id($pages, $siteId);
$breadcrumbs = sb_build_breadcrumbs($pagesById, $pageId);
$children = sb_get_children_pages($pages, $siteId, $pageId);
$parentId = (int)($page['parentId'] ?? 0);
$parentChildren = $parentId > 0 ? sb_get_children_pages($pages, $siteId, $parentId) : [];

// blocks for current page
$blocks = array_values(array_filter($blocksAll, fn($b) => (int)($b['pageId'] ?? 0) === $pageId));
usort($blocks, fn($a, $b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

// menu: take topMenuId for this site (if set), otherwise fallback to first menu
$menuItems = [];
foreach ($menusAll as $rec) {
    if ((int)($rec['siteId'] ?? 0) !== $siteId) continue;

    $menus = $rec['menus'] ?? [];
    if (!is_array($menus) || !count($menus)) break;

    $picked = null;

    if ($topMenuId > 0) {
        foreach ($menus as $m) {
            if ((int)($m['id'] ?? 0) === $topMenuId) { $picked = $m; break; }
        }
    }

    if (!$picked) $picked = $menus[0];

    $menuItems = $picked['items'] ?? [];
    if (!is_array($menuItems)) $menuItems = [];
    usort($menuItems, fn($a, $b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
    break;
}

if ($selectedMenu) {
    $menuItems = $selectedMenu['items'] ?? [];
    if (!is_array($menuItems)) $menuItems = [];
    usort($menuItems, fn($a,$b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
}
?>
<!doctype html>
<html lang="ru">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title><?=h($page['title'])?> — <?=h($site['name'])?></title>
  <style>
    body { font-family: Arial, sans-serif; margin: 0; background:#f6f7f8; color:#111; }
    .top {
      background:#fff;
      border-bottom:1px solid #e5e7ea;
      padding:12px 16px;
      display:flex;
      gap:10px;
      align-items:center;
      flex-wrap:wrap;
    }
    .content { padding: 18px; }
    .card { background:#fff; border:1px solid #e5e7ea; border-radius:12px; padding:16px; }
    a { color:#0b57d0; text-decoration:none; }
    a:hover { text-decoration:underline; }
    .muted { color:#6a737f; }
    .menu { display:flex; gap:10px; flex-wrap:wrap; }
    .menu a { padding:6px 10px; border-radius:999px; }
    .menu a.active { background:#eef2ff; font-weight:bold; }
    .block-text { margin-top:12px; line-height:1.6; font-size:16px; }
    .block-img { margin-top:14px; }
    .block-img img { max-width:100%; height:auto; border-radius:12px; border:1px solid #eee; display:block; }
    code { background:#f3f4f6; padding:2px 6px; border-radius:6px; }
    .btn { display:inline-block; padding:10px 14px; border-radius:12px; border:1px solid #e5e7ea; text-decoration:none; }
    .btn-primary { background:#2563eb; color:#fff; border-color:#2563eb; }
    .btn-secondary { background:#fff; color:#111; }

        /* columns2 */
    .cols2 {
    margin-top: 14px;
    display: grid;
    gap: 14px;
    }
    .cols2 .col {
    background: #fff;
    border: 1px solid #eee;
    border-radius: 12px;
    padding: 12px;
    }
    @media (max-width: 768px) {
    .cols2 { grid-template-columns: 1fr !important; }
    }

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

    .spacer { width:100%; }
    .spacerLine { height:1px; background:#e5e7ea; }

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

    .breadcrumbs { display:flex; gap:6px; flex-wrap:wrap; align-items:center; }
    .breadcrumbs a, .breadcrumbs span { font-size:12px; color:#6a737f; }
    .breadcrumbs a { color:#0b57d0; }
    .breadcrumbs .sep { color:#9aa3af; }

    .sectionNav {
        margin-top:14px;
        border:1px solid #e5e7ea;
        border-radius:14px;
        padding:12px;
        background:#fff;
    }
    .sectionNavTitle { font-weight:700; font-size:13px; }
    .sectionNavList { margin-top:10px; display:flex; gap:8px; flex-wrap:wrap; }
    .sectionNavList a {
        display:inline-block;
        padding:6px 10px;
        border-radius:999px;
        border:1px solid #e5e7ea;
        background:#fff;
        text-decoration:none;
    }
    .sectionNavList a.active { background:#eef2ff; border-color:#c7d2fe; font-weight:700; }
    .sectionNavHint { margin-top:8px; font-size:12px; color:#6a737f; }
  </style>
</head>
<body>
  <div class="top">
    <a href="/local/sitebuilder/index.php">← Назад</a>
    <div class="muted">/</div>
    <div><b><?=h($site['name'])?></b></div>
    <div class="muted">/</div>
    <div><?=h($page['title'])?></div>

    <div style="flex:1;"></div>

    <a href="/local/sitebuilder/editor.php?siteId=<?= (int)$siteId ?>&pageId=<?= (int)$pageId ?>" target="_blank">Редактор</a>
    <a href="/local/sitebuilder/menu.php?siteId=<?= (int)$siteId ?>" target="_blank">Меню</a>

        <div style="flex-basis:100%; height:0;"></div>

    <?php if ($breadcrumbs && count($breadcrumbs) > 1): ?>
    <div class="breadcrumbs">
        <?php foreach ($breadcrumbs as $i => $bc): ?>
        <?php
            $isLast = ($i === count($breadcrumbs)-1);
            $bcId = (int)($bc['id'] ?? 0);
            $bcTitle = (string)($bc['title'] ?? ('page#'.$bcId));
        ?>
        <?php if ($i > 0): ?><span class="sep">/</span><?php endif; ?>
        <?php if ($isLast): ?>
            <span><?= h($bcTitle) ?></span>
        <?php else: ?>
            <a href="<?= h(viewUrl($siteId, $bcId)) ?>"><?= h($bcTitle) ?></a>
        <?php endif; ?>
        <?php endforeach; ?>
    </div>
    <?php endif; ?>

    <div style="flex-basis:100%; height:0;"></div>

    <?php if ($menuItems): ?>
      <div class="menu">
        <?php foreach ($menuItems as $it): ?>
          <?php
            $type = (string)($it['type'] ?? '');
            $title = (string)($it['title'] ?? '');
            $isActive = false;
            $href = '#';

            if ($type === 'page') {
              $pid = (int)($it['pageId'] ?? 0);
              $href = viewUrl($siteId, $pid);
              $isActive = ($pid === $pageId);
              if ($title === '') $title = 'page#'.$pid;
            } elseif ($type === 'url') {
              $u = (string)($it['url'] ?? '');
              $href = $u !== '' ? $u : '#';
              if ($title === '') $title = $href;
            } else {
              $href = '#';
              if ($title === '') $title = '(unknown)';
            }
          ?>
          <a class="<?= $isActive ? 'active' : '' ?>" href="<?= h($href) ?>" <?= ($type === 'url' && preg_match('~^https?://~i', $href)) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
            <?= h($title) ?>
          </a>
        <?php endforeach; ?>
      </div>
    <?php else: ?>
      <!-- fallback: если меню ещё не создано -->
      <div class="menu">
        <?php foreach ($sitePages as $sp): ?>
          <?php $isActive = ((int)($sp['id'] ?? 0) === (int)$pageId); ?>
          <a class="<?= $isActive ? 'active' : '' ?>" href="<?= h(viewUrl($siteId, (int)($sp['id'] ?? 0))) ?>">
            <?= h($sp['title'] ?? '') ?>
          </a>
        <?php endforeach; ?>
      </div>
    <?php endif; ?>
  </div>

  <div class="content">
    <div class="card">
      <h1 style="margin-top:0;"><?=h($page['title'])?></h1>

      <?php
        // что показываем:
        // 1) если у текущей страницы есть дети -> показываем “Страницы раздела”
        // 2) иначе, если у страницы есть родитель -> показываем “В этом разделе” (соседи)
        ?>
        <?php if ($children && count($children) > 0): ?>
        <div class="sectionNav">
            <div class="sectionNavTitle">Страницы раздела</div>
            <div class="sectionNavList">
            <?php foreach ($children as $ch): ?>
                <?php $chId = (int)($ch['id'] ?? 0); ?>
                <a href="<?= h(viewUrl($siteId, $chId)) ?>">
                <?= h((string)($ch['title'] ?? ('page#'.$chId))) ?>
                </a>
            <?php endforeach; ?>
            </div>
            <div class="sectionNavHint">Раздел — это страница, у которой есть дочерние страницы (parentId = <?= (int)$pageId ?>).</div>
        </div>
        <?php elseif ($parentId > 0 && $parentChildren && count($parentChildren) > 0): ?>
        <div class="sectionNav">
            <div class="sectionNavTitle">В этом разделе</div>
            <div class="sectionNavList">
            <?php foreach ($parentChildren as $ch): ?>
                <?php $chId = (int)($ch['id'] ?? 0); $active = ($chId === $pageId); ?>
                <a class="<?= $active ? 'active' : '' ?>" href="<?= h(viewUrl($siteId, $chId)) ?>">
                <?= h((string)($ch['title'] ?? ('page#'.$chId))) ?>
                </a>
            <?php endforeach; ?>
            </div>
            <div class="sectionNavHint">Ты внутри раздела (parentId = <?= (int)$parentId ?>). Тут показаны “соседи”.</div>
        </div>
        <?php endif; ?>

      <?php if (!$blocks): ?>
        <div class="muted">Блоков пока нет. Добавь их в редакторе.</div>
      <?php else: ?>
        <?php foreach ($blocks as $b): ?>
          <?php $type = (string)($b['type'] ?? ''); ?>

          <?php if ($type === 'text'): ?>
            <div class="block-text">
              <?= nl2br(h((string)($b['content']['text'] ?? ''))) ?>
            </div>
          <?php endif; ?>

          <?php if ($type === 'button'): ?>
			  <?php
				$text = (string)($b['content']['text'] ?? '');
				$url = (string)($b['content']['url'] ?? '#');
				$variant = (string)($b['content']['variant'] ?? 'primary');
				$cls = ($variant === 'secondary') ? 'btn btn-secondary' : 'btn btn-primary';
			  ?>
			  <div class="block-btn" style="margin-top:14px;">
				<a class="<?=h($cls)?>" href="<?=h($url)?>" <?= preg_match('~^https?://~i', $url) ? 'target="_blank" rel="noopener noreferrer"' : '' ?>>
				  <?=h($text)?>
				</a>
			  </div>
			<?php endif; ?>

          <?php if ($type === 'image'): ?>
            <?php $fileId = (int)($b['content']['fileId'] ?? 0); ?>
            <?php $alt = (string)($b['content']['alt'] ?? ''); ?>
            <?php if ($fileId > 0): ?>
              <div class="block-img">
                <img src="<?= h(downloadUrl($siteId, $fileId)) ?>" alt="<?= h($alt) ?>">
              </div>
            <?php else: ?>
              <div class="muted" style="margin-top:12px;">(image) файл не выбран</div>
            <?php endif; ?>
          <?php endif; ?>

          <?php if ($type === 'heading'): ?>

          <?php
            $text = (string)($b['content']['text'] ?? '');
            $level = (string)($b['content']['level'] ?? 'h2');
            $align = (string)($b['content']['align'] ?? 'left');
            if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
            if (!in_array($align, ['left','center','right'], true)) $align = 'left';
          ?>
            <<?=h($level)?> style="margin-top:16px; text-align:<?=h($align)?>;">
                <?=h($text)?>
            </<?=h($level)?>>
            <?php endif; ?>


          <?php if ($type === 'columns2'): ?>
            <?php
                $left  = (string)($b['content']['left'] ?? '');
                $right = (string)($b['content']['right'] ?? '');
                $ratio = (string)($b['content']['ratio'] ?? '50-50');

                if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

                $tpl = '1fr 1fr';
                if ($ratio === '33-67') $tpl = '1fr 2fr';
                if ($ratio === '67-33') $tpl = '2fr 1fr';
            ?>
            <div class="cols2" style="grid-template-columns: <?=h($tpl)?>;">
                <div class="col"><?= nl2br(h($left)) ?></div>
                <div class="col"><?= nl2br(h($right)) ?></div>
            </div>
            <?php endif; ?>

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


            <?php if ($type === 'spacer'): ?>
                <?php
                    $h = (int)($b['content']['height'] ?? 40);
                    if ($h < 10) $h = 10;
                    if ($h > 200) $h = 200;
                    $line = (bool)($b['content']['line'] ?? false);
                ?>
                <div class="spacer" style="height: <?= (int)$h ?>px; position:relative; margin-top:14px;">
                    <?php if ($line): ?>
                    <div class="spacerLine" style="position:absolute; left:0; right:0; top:50%;"></div>
                    <?php endif; ?>
                </div>
            <?php endif; ?>

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

        <?php endforeach; ?>
      <?php endif; ?>

      <div style="margin-top:16px;" class="muted">
        <div><b>slug:</b> <code><?=h($page['slug'])?></code></div>
        <div><b>pageId:</b> <?= (int)$page['id'] ?> &nbsp; <b>siteId:</b> <?= (int)$site['id'] ?></div>
      </div>
    </div>
  </div>
</body>
</html>
