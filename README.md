Давай. Сначала сделаем 2 вещи сразу:

хлебные крошки над h1

более аккуратный стиль левого меню


Новый файл не нужен — достаточно внести правки в текущий public.php.


---

1. Добавить функции для breadcrumbs

Куда вставить

В public.php после функции:

function sb_page_ancestor_ids(...)

Вставь это:

function sb_page_breadcrumbs(array $pages, int $pageId): array {
    $byId = [];
    foreach ($pages as $p) {
        $byId[(int)($p['id'] ?? 0)] = $p;
    }

    $chain = [];
    $current = $pageId;
    $guard = 0;

    while ($current > 0 && isset($byId[$current]) && $guard < 100) {
        array_unshift($chain, $byId[$current]);
        $current = (int)($byId[$current]['parentId'] ?? 0);
        $guard++;
    }

    return $chain;
}

function sb_render_breadcrumbs(array $items, array $site): string {
    if (!$items) return '';

    $parts = [];
    $lastIndex = count($items) - 1;

    foreach ($items as $i => $p) {
        $title = (string)($p['title'] ?? 'Page');

        if ($i === $lastIndex) {
            $parts[] = '<span class="crumb current">' . h($title) . '</span>';
        } else {
            $parts[] = '<a class="crumb" href="' . h(public_page_url($site, $p)) . '">' . h($title) . '</a>';
        }
    }

    return '<nav class="breadcrumbs">' . implode('<span class="crumbSep">/</span>', $parts) . '</nav>';
}


---

2. Собрать breadcrumbs

Куда вставить

Найди этот блок:

$leftMenuHtml = '';
if (!empty($layout['showLeft']) && $leftMode === 'menu') {
    $publishedPages = sb_site_published_pages($siteId);
    $tree = sb_pages_build_tree($publishedPages);
    $activePathIds = sb_page_ancestor_ids($publishedPages, $pageId);
    $leftMenuHtml = sb_render_left_menu_tree($tree, $site, $pageId, $activePathIds);
}

И сразу после него вставь:

$publishedPagesForNav = sb_site_published_pages($siteId);
$breadcrumbsItems = sb_page_breadcrumbs($publishedPagesForNav, $pageId);
$breadcrumbsHtml = sb_render_breadcrumbs($breadcrumbsItems, $site);


---

3. Вывести breadcrumbs над h1

Куда вставить

Найди в центре страницы этот блок:

<div class="hero">
  <h1><?=h($page['title'] ?? '')?></h1>
  <div class="muted"><?=h($site['name'] ?? 'Site')?></div>
</div>

Замени на:

<div class="hero">
  <?php if ($breadcrumbsHtml !== ''): ?>
    <?= $breadcrumbsHtml ?>
  <?php endif; ?>

  <h1><?=h($page['title'] ?? '')?></h1>
  <div class="muted"><?=h($site['name'] ?? 'Site')?></div>
</div>


---

4. Улучшить стили breadcrumbs

Куда вставить

В <style> добавь:

.breadcrumbs{
  display:flex;
  flex-wrap:wrap;
  align-items:center;
  gap:8px;
  margin-bottom:10px;
  font-size:13px;
  color:var(--muted);
}

.crumb{
  color:var(--muted);
  text-decoration:none;
}

.crumb:hover{
  color:var(--sb-accent);
  text-decoration:none;
}

.crumb.current{
  color:var(--text);
  font-weight:600;
}

.crumbSep{
  color:#c0c7d1;
}


---

5. Улучшить стиль левого меню

Найди в <style> текущие стили:

.sideTree{
  display:flex;
  flex-direction:column;
  gap:6px;
}

.sideTreeChildren{
  margin-top:6px;
  margin-left:14px;
  padding-left:10px;
  border-left:1px dashed #dbe1e7;
}

.sideTreeNode{
  min-width:0;
}

.sideMenuLink{
  display:flex;
  align-items:flex-start;
  gap:8px;
  width:100%;
  padding:10px 12px;
  border-radius:12px;
  border:1px solid var(--line);
  background:#fff;
  color:var(--text);
  text-decoration:none;
  line-height:1.35;
}

.sideMenuLink:hover{
  text-decoration:none;
  border-color:#d1d5db;
  background:#f9fafb;
}

.sideMenuLink.active{
  background: color-mix(in srgb, var(--sb-accent) 12%, #fff);
  border-color: color-mix(in srgb, var(--sb-accent) 28%, #e5e7eb);
  color: var(--sb-accent);
  font-weight:700;
}

.sideMenuLink.open{
  border-color:#d6dbe1;
}

.sideMenuCaret{
  width:14px;
  flex:0 0 14px;
  color:#64748b;
  text-align:center;
  line-height:1.3;
  margin-top:1px;
}

.sideMenuCaretEmpty{
  visibility:hidden;
}

.sideMenuText{
  min-width:0;
  flex:1 1 auto;
  word-break:break-word;
}

И замени на это:

.sideTree{
  display:flex;
  flex-direction:column;
  gap:4px;
}

.sideTreeChildren{
  margin-top:4px;
  margin-left:12px;
  padding-left:10px;
  border-left:1px dashed #dbe1e7;
}

.sideTreeNode{
  min-width:0;
}

.sideMenuLink{
  display:flex;
  align-items:center;
  gap:8px;
  width:100%;
  padding:9px 10px;
  border-radius:10px;
  border:1px solid transparent;
  background:transparent;
  color:var(--text);
  text-decoration:none;
  line-height:1.3;
  transition:background .15s ease, border-color .15s ease, color .15s ease;
}

.sideMenuLink:hover{
  text-decoration:none;
  background:#f8fafc;
  border-color:#e5e7eb;
}

.sideMenuLink.active{
  background: color-mix(in srgb, var(--sb-accent) 10%, #fff);
  border-color: color-mix(in srgb, var(--sb-accent) 20%, #e5e7eb);
  color: var(--sb-accent);
  font-weight:700;
}

.sideMenuLink.open{
  background:#fcfcfd;
}

.sideMenuCaret{
  width:14px;
  flex:0 0 14px;
  color:#64748b;
  text-align:center;
  line-height:1;
  font-size:11px;
}

.sideMenuCaretEmpty{
  visibility:hidden;
}

.sideMenuText{
  min-width:0;
  flex:1 1 auto;
  word-break:break-word;
}

.sideTree.level-0 > .sideTreeNode > .sideMenuLink{
  font-weight:600;
}

.sideTree.level-1 > .sideTreeNode > .sideMenuLink,
.sideTree.level-2 > .sideTreeNode > .sideMenuLink,
.sideTree.level-3 > .sideTreeNode > .sideMenuLink{
  font-weight:400;
  font-size:14px;
}


---

Что получится

После этого:

над заголовком будет путь страницы

левое меню станет визуально чище

вложенность будет читаться лучше

активная страница будет смотреться аккуратнее



---

Если захочешь следующий шаг

После этого я бы сделал раскрытие/сворачивание веток левого меню по клику на стрелку, чтобы меню было уже совсем живым.