Отлично. Раз дерево страниц уже работает в editor.php, следующий правильный шаг — протащить эту иерархию в public.php:

breadcrumbs

левую навигацию по разделу

список дочерних страниц у текущей страницы

нормальное понимание, где пользователь находится


Идём пакетом.


---

Что сейчас сделаем

Нужно заменить 2 файла:

1. /local/sitebuilder/lib/public_render.php


2. /local/sitebuilder/views/layout/public_page.php




---

1. Замени lib/public_render.php

Полностью замени файл:

/local/sitebuilder/lib/public_render.php

на этот код:

<?php

require_once __DIR__ . '/json.php';
require_once __DIR__ . '/helpers.php';

if (!function_exists('sb_public_h')) {
    function sb_public_h(string $s): string
    {
        return htmlspecialchars($s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    }
}

if (!function_exists('sb_public_find_site')) {
    function sb_public_find_site(int $siteId): ?array
    {
        foreach (sb_read_sites() as $site) {
            if ((int)($site['id'] ?? 0) === $siteId) {
                return $site;
            }
        }
        return null;
    }
}

if (!function_exists('sb_public_pages_for_site')) {
    function sb_public_pages_for_site(int $siteId): array
    {
        $pages = array_values(array_filter(sb_read_pages(), static function ($p) use ($siteId) {
            return (int)($p['siteId'] ?? 0) === $siteId;
        }));

        usort($pages, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
            if ($sortCmp !== 0) {
                return $sortCmp;
            }
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return array_map('sb_normalize_page_record', $pages);
    }
}

if (!function_exists('sb_public_find_page_for_site')) {
    function sb_public_find_page_for_site(int $siteId, int $pageId): ?array
    {
        foreach (sb_read_pages() as $page) {
            if (
                (int)($page['siteId'] ?? 0) === $siteId &&
                (int)($page['id'] ?? 0) === $pageId
            ) {
                return sb_normalize_page_record($page);
            }
        }
        return null;
    }
}

if (!function_exists('sb_public_page_blocks')) {
    function sb_public_page_blocks(int $pageId): array
    {
        $blocks = array_values(array_filter(sb_read_blocks(), static function ($b) use ($pageId) {
            return (int)($b['pageId'] ?? 0) === $pageId;
        }));

        usort($blocks, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
            if ($sortCmp !== 0) {
                return $sortCmp;
            }
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return array_map('sb_normalize_block_record', $blocks);
    }
}

if (!function_exists('sb_public_layout_for_site')) {
    function sb_public_layout_for_site(int $siteId): array
    {
        foreach (sb_read_layouts() as $layout) {
            if ((int)($layout['siteId'] ?? 0) === $siteId) {
                return sb_normalize_layout_record($layout);
            }
        }

        return sb_normalize_layout_record(sb_layout_default_record($siteId));
    }
}

if (!function_exists('sb_public_menu_for_site')) {
    function sb_public_menu_for_site(array $site): ?array
    {
        $topMenuId = (int)($site['topMenuId'] ?? 0);
        if ($topMenuId <= 0) {
            return null;
        }

        foreach (sb_read_menus() as $menu) {
            if ((int)($menu['id'] ?? 0) === $topMenuId) {
                return sb_normalize_menu_record($menu);
            }
        }

        return null;
    }
}

if (!function_exists('sb_public_page_url')) {
    function sb_public_page_url(string $basePath, int $siteId, int $pageId): string
    {
        return $basePath . '/public.php?siteId=' . $siteId . '&pageId=' . $pageId;
    }
}

if (!function_exists('sb_public_menu_item_url')) {
    function sb_public_menu_item_url(array $item, string $basePath, int $siteId): string
    {
        $type = (string)($item['type'] ?? 'page');

        if ($type === 'page') {
            $pageId = (int)($item['pageId'] ?? 0);
            return $pageId > 0 ? sb_public_page_url($basePath, $siteId, $pageId) : '#';
        }

        $url = trim((string)($item['url'] ?? ''));
        return $url !== '' ? $url : '#';
    }
}

if (!function_exists('sb_public_render_block')) {
    function sb_public_render_block(array $block, array $context = []): string
    {
        $type = (string)($block['type'] ?? 'text');
        $template = dirname(__DIR__) . '/views/blocks/' . $type . '.php';

        if (!file_exists($template)) {
            $template = dirname(__DIR__) . '/views/blocks/text.php';
        }

        $block = sb_normalize_block_record($block);
        $content = (array)($block['content'] ?? []);
        $props = (array)($block['props'] ?? []);

        ob_start();
        include $template;
        return (string)ob_get_clean();
    }
}

if (!function_exists('sb_public_render_blocks')) {
    function sb_public_render_blocks(array $blocks, array $context = []): string
    {
        if (!$blocks) {
            return '';
        }

        $html = '';
        foreach ($blocks as $block) {
            $html .= sb_public_render_block($block, $context);
        }
        return $html;
    }
}

if (!function_exists('sb_public_render_menu')) {
    function sb_public_render_menu(?array $menu, string $basePath, int $siteId): string
    {
        if (!$menu || empty($menu['items']) || !is_array($menu['items'])) {
            return '';
        }

        $html = '<nav class="sb-public-menu">';
        foreach ($menu['items'] as $item) {
            $title = sb_public_h((string)($item['title'] ?? 'Пункт'));
            $url = sb_public_h(sb_public_menu_item_url($item, $basePath, $siteId));
            $target = sb_public_h((string)($item['target'] ?? '_self'));
            $html .= '<a class="sb-public-menu__link" href="' . $url . '" target="' . $target . '">' . $title . '</a>';
        }
        $html .= '</nav>';

        return $html;
    }
}

if (!function_exists('sb_public_build_page_map')) {
    function sb_public_build_page_map(array $pages): array
    {
        $map = [];
        foreach ($pages as $page) {
            $page = sb_normalize_page_record($page);
            $page['children'] = [];
            $map[(int)$page['id']] = $page;
        }

        foreach ($map as $id => $page) {
            $parentId = (int)($page['parentId'] ?? 0);
            if ($parentId > 0 && isset($map[$parentId])) {
                $map[$parentId]['children'][] = $id;
            }
        }

        return $map;
    }
}

if (!function_exists('sb_public_page_children')) {
    function sb_public_page_children(array $pages, int $parentId): array
    {
        $result = [];

        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) === $parentId) {
                $result[] = sb_normalize_page_record($page);
            }
        }

        usort($result, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
            if ($sortCmp !== 0) {
                return $sortCmp;
            }
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $result;
    }
}

if (!function_exists('sb_public_breadcrumbs')) {
    function sb_public_breadcrumbs(array $pages, ?array $currentPage): array
    {
        if (!$currentPage) {
            return [];
        }

        $map = [];
        foreach ($pages as $page) {
            $page = sb_normalize_page_record($page);
            $map[(int)$page['id']] = $page;
        }

        $chain = [];
        $cursor = $currentPage;
        $safety = 0;

        while ($cursor && $safety < 1000) {
            array_unshift($chain, $cursor);
            $parentId = (int)($cursor['parentId'] ?? 0);
            if ($parentId <= 0 || !isset($map[$parentId])) {
                break;
            }
            $cursor = $map[$parentId];
            $safety++;
        }

        return $chain;
    }
}

if (!function_exists('sb_public_section_root')) {
    function sb_public_section_root(array $pages, ?array $currentPage): ?array
    {
        if (!$currentPage) {
            return null;
        }

        $map = [];
        foreach ($pages as $page) {
            $page = sb_normalize_page_record($page);
            $map[(int)$page['id']] = $page;
        }

        $cursor = $currentPage;
        $last = $cursor;
        $safety = 0;

        while ($cursor && $safety < 1000) {
            $parentId = (int)($cursor['parentId'] ?? 0);
            if ($parentId <= 0 || !isset($map[$parentId])) {
                break;
            }
            $last = $map[$parentId];
            $cursor = $map[$parentId];
            $safety++;
        }

        return $last;
    }
}

if (!function_exists('sb_public_render_breadcrumbs')) {
    function sb_public_render_breadcrumbs(array $breadcrumbs, string $basePath, int $siteId): string
    {
        if (!$breadcrumbs) {
            return '';
        }

        $html = '<nav class="sb-breadcrumbs">';
        $lastIndex = count($breadcrumbs) - 1;

        foreach ($breadcrumbs as $i => $page) {
            $title = sb_public_h((string)($page['title'] ?? 'Страница'));
            if ($i < $lastIndex) {
                $url = sb_public_h(sb_public_page_url($basePath, $siteId, (int)$page['id']));
                $html .= '<a class="sb-breadcrumbs__link" href="' . $url . '">' . $title . '</a>';
                $html .= '<span class="sb-breadcrumbs__sep">/</span>';
            } else {
                $html .= '<span class="sb-breadcrumbs__current">' . $title . '</span>';
            }
        }

        $html .= '</nav>';
        return $html;
    }
}

if (!function_exists('sb_public_render_section_nav')) {
    function sb_public_render_section_nav(array $pages, ?array $currentPage, string $basePath, int $siteId): string
    {
        if (!$currentPage) {
            return '';
        }

        $sectionRoot = sb_public_section_root($pages, $currentPage);
        if (!$sectionRoot) {
            return '';
        }

        $rootId = (int)($sectionRoot['id'] ?? 0);
        $children = sb_public_page_children($pages, $rootId);

        $html = '<div class="sb-section-nav">';
        $html .= '<div class="sb-section-nav__title">' . sb_public_h((string)($sectionRoot['title'] ?? 'Раздел')) . '</div>';

        if ($children) {
            $html .= '<div class="sb-section-nav__list">';

            if ((int)($currentPage['id'] ?? 0) === $rootId) {
                $html .= '<a class="sb-section-nav__link is-active" href="' . sb_public_h(sb_public_page_url($basePath, $siteId, $rootId)) . '">' . sb_public_h((string)($sectionRoot['title'] ?? 'Раздел')) . '</a>';
            } else {
                $html .= '<a class="sb-section-nav__link" href="' . sb_public_h(sb_public_page_url($basePath, $siteId, $rootId)) . '">' . sb_public_h((string)($sectionRoot['title'] ?? 'Раздел')) . '</a>';
            }

            foreach ($children as $child) {
                $active = (int)($child['id'] ?? 0) === (int)($currentPage['id'] ?? 0) ? ' is-active' : '';
                $html .= '<a class="sb-section-nav__link' . $active . '" href="' . sb_public_h(sb_public_page_url($basePath, $siteId, (int)$child['id'])) . '">' . sb_public_h((string)($child['title'] ?? 'Страница')) . '</a>';
            }

            $html .= '</div>';
        }

        $html .= '</div>';
        return $html;
    }
}

if (!function_exists('sb_public_render_child_pages')) {
    function sb_public_render_child_pages(array $pages, ?array $currentPage, string $basePath, int $siteId): string
    {
        if (!$currentPage) {
            return '';
        }

        $children = sb_public_page_children($pages, (int)($currentPage['id'] ?? 0));
        if (!$children) {
            return '';
        }

        $html = '<section class="sb-child-pages">';
        $html .= '<h2 class="sb-child-pages__title">Разделы</h2>';
        $html .= '<div class="sb-child-pages__grid">';

        foreach ($children as $child) {
            $html .= '<a class="sb-child-pages__card" href="' . sb_public_h(sb_public_page_url($basePath, $siteId, (int)$child['id'])) . '">';
            $html .= '<div class="sb-child-pages__name">' . sb_public_h((string)($child['title'] ?? 'Страница')) . '</div>';
            $html .= '<div class="sb-child-pages__slug">' . sb_public_h((string)($child['slug'] ?? '')) . '</div>';
            $html .= '</a>';
        }

        $html .= '</div>';
        $html .= '</section>';

        return $html;
    }
}

if (!function_exists('sb_public_build_view_model')) {
    function sb_public_build_view_model(int $siteId, ?int $requestedPageId, string $basePath): ?array
    {
        $site = sb_public_find_site($siteId);
        if (!$site) {
            return null;
        }

        $pages = sb_public_pages_for_site($siteId);
        $currentPage = null;

        if ($requestedPageId && $requestedPageId > 0) {
            $currentPage = sb_public_find_page_for_site($siteId, $requestedPageId);
        }

        if (!$currentPage) {
            $homePageId = (int)($site['homePageId'] ?? 0);
            if ($homePageId > 0) {
                $currentPage = sb_public_find_page_for_site($siteId, $homePageId);
            }
        }

        if (!$currentPage && !empty($pages)) {
            $currentPage = $pages[0];
        }

        $layout = sb_public_layout_for_site($siteId);
        $menu = sb_public_menu_for_site($site);
        $pageBlocks = $currentPage ? sb_public_page_blocks((int)$currentPage['id']) : [];

        $settings = isset($site['settings']) && is_array($site['settings']) ? $site['settings'] : [];
        $layoutSettings = isset($layout['settings']) && is_array($layout['settings']) ? $layout['settings'] : [];

        $containerWidth = max(320, min(1920, (int)($settings['containerWidth'] ?? 1360)));
        $accent = (string)($settings['accent'] ?? '#2563eb');
        if (!preg_match('/^#[0-9a-fA-F]{6}$/', $accent)) {
            $accent = '#2563eb';
        }

        $breadcrumbs = sb_public_breadcrumbs($pages, $currentPage);
        $sectionNavHtml = sb_public_render_section_nav($pages, $currentPage, $basePath, $siteId);
        $childPagesHtml = sb_public_render_child_pages($pages, $currentPage, $basePath, $siteId);

        return [
            'site' => $site,
            'pages' => $pages,
            'currentPage' => $currentPage,
            'pageBlocks' => $pageBlocks,
            'layout' => $layout,
            'menu' => $menu,
            'basePath' => $basePath,
            'siteId' => $siteId,
            'containerWidth' => $containerWidth,
            'accent' => $accent,
            'showHeader' => !empty($layoutSettings['showHeader']),
            'showFooter' => !empty($layoutSettings['showFooter']),
            'showLeft' => !empty($layoutSettings['showLeft']),
            'showRight' => !empty($layoutSettings['showRight']),
            'leftWidth' => max(120, min(800, (int)($layoutSettings['leftWidth'] ?? 260))),
            'rightWidth' => max(120, min(800, (int)($layoutSettings['rightWidth'] ?? 260))),
            'leftMode' => (string)($layoutSettings['leftMode'] ?? 'blocks'),
            'breadcrumbs' => $breadcrumbs,
            'breadcrumbsHtml' => sb_public_render_breadcrumbs($breadcrumbs, $basePath, $siteId),
            'sectionNavHtml' => $sectionNavHtml,
            'childPagesHtml' => $childPagesHtml,
        ];
    }
}


---

2. Замени views/layout/public_page.php

Полностью замени файл:

/local/sitebuilder/views/layout/public_page.php

на этот код:

<?php
/** @var array $vm */

$site = $vm['site'];
$pages = $vm['pages'];
$currentPage = $vm['currentPage'];
$pageBlocks = $vm['pageBlocks'];
$layout = $vm['layout'];
$menu = $vm['menu'];
$basePath = $vm['basePath'];
$siteId = (int)$vm['siteId'];

$headerBlocks = $layout['zones']['header'] ?? [];
$footerBlocks = $layout['zones']['footer'] ?? [];
$leftBlocks = $layout['zones']['left'] ?? [];
$rightBlocks = $layout['zones']['right'] ?? [];

$headerHtml = sb_public_render_blocks($headerBlocks, $vm);
$footerHtml = sb_public_render_blocks($footerBlocks, $vm);
$leftHtml = sb_public_render_blocks($leftBlocks, $vm);
$rightHtml = sb_public_render_blocks($rightBlocks, $vm);
$pageHtml = sb_public_render_blocks($pageBlocks, $vm);
$menuHtml = sb_public_render_menu($menu, $basePath, $siteId);

$leftContentHtml = $vm['leftMode'] === 'menu' && $menuHtml !== '' ? $menuHtml : $leftHtml;

if ($vm['leftMode'] === 'menu' && $vm['sectionNavHtml'] !== '') {
    $leftContentHtml = $vm['sectionNavHtml'];
}
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title><?= sb_public_h((string)($currentPage['title'] ?? $site['name'] ?? 'SiteBuilder')) ?></title>
    <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css">
    <style>
        :root {
            --sb-accent: <?= sb_public_h($vm['accent']) ?>;
            --sb-container-width: <?= (int)$vm['containerWidth'] ?>px;
            --sb-left-width: <?= (int)$vm['leftWidth'] ?>px;
            --sb-right-width: <?= (int)$vm['rightWidth'] ?>px;
        }
    </style>
</head>
<body>
<div class="sb-public-shell">
    <?php if ($vm['showHeader']): ?>
        <header class="sb-public-header">
            <div class="sb-container">
                <?php if ($headerHtml !== ''): ?>
                    <?= $headerHtml ?>
                <?php else: ?>
                    <div class="sb-brand">
                        <?= sb_public_h((string)($site['name'] ?? 'SiteBuilder')) ?>
                    </div>
                <?php endif; ?>

                <?php if ($menuHtml !== ''): ?>
                    <?= $menuHtml ?>
                <?php elseif (!empty($pages)): ?>
                    <nav class="sb-public-menu">
                        <?php foreach ($pages as $page): ?>
                            <?php if ((int)($page['parentId'] ?? 0) !== 0) continue; ?>
                            <a class="sb-public-menu__link" href="<?= sb_public_h(sb_public_page_url($basePath, $siteId, (int)$page['id'])) ?>">
                                <?= sb_public_h((string)($page['title'] ?? 'Страница')) ?>
                            </a>
                        <?php endforeach; ?>
                    </nav>
                <?php endif; ?>
            </div>
        </header>
    <?php endif; ?>

    <main class="sb-public-main">
        <div class="sb-container">
            <?php if (!empty($vm['breadcrumbsHtml'])): ?>
                <?= $vm['breadcrumbsHtml'] ?>
            <?php endif; ?>

            <div class="sb-layout <?= $vm['showLeft'] ? 'sb-layout--left' : '' ?> <?= $vm['showRight'] ? 'sb-layout--right' : '' ?>">
                <?php if ($vm['showLeft']): ?>
                    <aside class="sb-sidebar sb-sidebar--left">
                        <div class="sb-box">
                            <?= $leftContentHtml !== '' ? $leftContentHtml : '<div class="sb-empty">Левая зона пуста</div>' ?>
                        </div>
                    </aside>
                <?php endif; ?>

                <section class="sb-content">
                    <div class="sb-box sb-box--content">
                        <?php if ($currentPage): ?>
                            <h1 class="sb-page-title">
                                <?= sb_public_h((string)($currentPage['title'] ?? 'Страница')) ?>
                            </h1>

                            <?php if (!empty($vm['childPagesHtml'])): ?>
                                <?= $vm['childPagesHtml'] ?>
                            <?php endif; ?>

                            <?= $pageHtml !== '' ? $pageHtml : '<div class="sb-empty">На странице пока нет блоков</div>' ?>
                        <?php else: ?>
                            <div class="sb-empty">У сайта пока нет страниц</div>
                        <?php endif; ?>
                    </div>
                </section>

                <?php if ($vm['showRight']): ?>
                    <aside class="sb-sidebar sb-sidebar--right">
                        <div class="sb-box">
                            <?= $rightHtml !== '' ? $rightHtml : '<div class="sb-empty">Правая зона пуста</div>' ?>
                        </div>
                    </aside>
                <?php endif; ?>
            </div>
        </div>
    </main>

    <?php if ($vm['showFooter']): ?>
        <footer class="sb-public-footer">
            <div class="sb-container">
                <?= $footerHtml !== '' ? $footerHtml : '<div class="sb-footer-note">© ' . date('Y') . ' ' . sb_public_h((string)($site['name'] ?? 'SiteBuilder')) . '</div>' ?>
            </div>
        </footer>
    <?php endif; ?>
</div>
</body>
</html>


---

3. Что ещё желательно добавить в public.css

Чтобы breadcrumbs и section-nav выглядели нормально, добавь в конец файла:

/local/sitebuilder/assets/public/public.css

вот этот CSS:

.sb-breadcrumbs {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    align-items: center;
    margin-bottom: 16px;
    font-size: 14px;
    color: #6b7280;
}

.sb-breadcrumbs__link {
    color: var(--sb-accent);
    text-decoration: none;
}

.sb-breadcrumbs__sep {
    color: #9ca3af;
}

.sb-breadcrumbs__current {
    color: #111827;
    font-weight: 600;
}

.sb-section-nav__title {
    font-size: 14px;
    font-weight: 700;
    margin-bottom: 12px;
    color: #111827;
}

.sb-section-nav__list {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-section-nav__link {
    display: block;
    text-decoration: none;
    color: #374151;
    padding: 10px 12px;
    border-radius: 10px;
    background: #f9fafb;
    border: 1px solid #e5e7eb;
}

.sb-section-nav__link.is-active {
    color: var(--sb-accent);
    background: #eef2ff;
    border-color: #c7d2fe;
    font-weight: 600;
}

.sb-child-pages {
    margin: 0 0 24px;
}

.sb-child-pages__title {
    margin: 0 0 14px;
    font-size: 20px;
    line-height: 1.2;
}

.sb-child-pages__grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(220px, 1fr));
    gap: 14px;
}

.sb-child-pages__card {
    display: block;
    text-decoration: none;
    color: inherit;
    padding: 16px;
    border-radius: 14px;
    background: #f9fafb;
    border: 1px solid #e5e7eb;
    transition: transform .15s ease, box-shadow .15s ease, border-color .15s ease;
}

.sb-child-pages__card:hover {
    transform: translateY(-1px);
    border-color: #c7d2fe;
    box-shadow: 0 4px 14px rgba(37, 99, 235, 0.08);
}

.sb-child-pages__name {
    font-weight: 700;
    margin-bottom: 6px;
    color: #111827;
}

.sb-child-pages__slug {
    color: #6b7280;
    font-size: 13px;
}


---

Что проверить

После замены:

1. открой родительскую страницу, у которой есть дочерние


2. сверху должны появиться breadcrumbs


3. в контенте должен появиться блок Разделы с дочерними страницами


4. если в layout включена левая колонка и leftMode = menu, слева должна быть section-navigation


5. переход по дочерним страницам должен работать




---

Что дальше логично делать

Теперь у тебя уже есть:

дерево страниц в editor

breadcrumbs в public

section navigation в public

дочерние страницы как разделы


Следующий логичный шаг — такую же иерархию протащить в menu.php, чтобы в селекте страниц и в меню было видно дерево, а не плоский список.