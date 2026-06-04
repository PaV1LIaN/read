Заменяй полностью:

/local/sitebuilder/views/layout/public_page.php

на этот файл:

<?php
/** @var array $vm */

if (!function_exists('sb_public_h')) {
    function sb_public_h(string $s): string
    {
        return htmlspecialchars($s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    }
}

$site = is_array($vm['site'] ?? null) ? $vm['site'] : [];
$pages = is_array($vm['pages'] ?? null) ? $vm['pages'] : [];
$currentPage = is_array($vm['currentPage'] ?? null) ? $vm['currentPage'] : null;
$pageBlocks = is_array($vm['pageBlocks'] ?? null) ? $vm['pageBlocks'] : [];
$layout = is_array($vm['layout'] ?? null) ? $vm['layout'] : [];
$menu = $vm['menu'] ?? null;
$basePath = (string)($vm['basePath'] ?? '');
$siteId = (int)($vm['siteId'] ?? 0);

if (!function_exists('sb_public_to_array')) {
    function sb_public_to_array($value): array
    {
        if (is_array($value)) {
            return $value;
        }

        if (is_string($value) && trim($value) !== '') {
            $decoded = json_decode($value, true);

            if (is_array($decoded)) {
                return $decoded;
            }
        }

        return [];
    }
}

if (!function_exists('sb_public_clamp_int')) {
    function sb_public_clamp_int($value, int $min, int $max): int
    {
        $value = (int)$value;

        if ($value < $min) {
            return $min;
        }

        if ($value > $max) {
            return $max;
        }

        return $value;
    }
}

/* =========================================================
   AUTO MENU
   ========================================================= */

if (!function_exists('sb_public_auto_menu_is_page_visible')) {
    function sb_public_auto_menu_is_page_visible(array $page, int $currentPageId = 0): bool
    {
        $status = (string)($page['status'] ?? 'published');
        $pageId = (int)($page['id'] ?? 0);

        if ($pageId === $currentPageId) {
            return true;
        }

        return $status === 'published';
    }
}

if (!function_exists('sb_public_auto_menu_children')) {
    function sb_public_auto_menu_children(array $pages, int $parentId, int $currentPageId = 0): array
    {
        $items = [];

        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) !== $parentId) {
                continue;
            }

            if (!sb_public_auto_menu_is_page_visible($page, $currentPageId)) {
                continue;
            }

            $items[] = $page;
        }

        usort($items, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

            if ($sortCmp !== 0) {
                return $sortCmp;
            }

            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $items;
    }
}

if (!function_exists('sb_public_auto_menu_has_active_child')) {
    function sb_public_auto_menu_has_active_child(array $pages, int $pageId, int $currentPageId): bool
    {
        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) !== $pageId) {
                continue;
            }

            $childId = (int)($page['id'] ?? 0);

            if ($childId === $currentPageId) {
                return true;
            }

            if (sb_public_auto_menu_has_active_child($pages, $childId, $currentPageId)) {
                return true;
            }
        }

        return false;
    }
}

if (!function_exists('sb_public_render_auto_menu_level')) {
    function sb_public_render_auto_menu_level(array $pages, int $parentId, string $basePath, int $siteId, int $currentPageId, int $level = 0): string
    {
        $children = sb_public_auto_menu_children($pages, $parentId, $currentPageId);

        if (empty($children)) {
            return '';
        }

        $class = $level === 0 ? 'sb-public-menu' : 'sb-public-menu__dropdown';
        $html = '<nav class="' . $class . '">';

        foreach ($children as $page) {
            $pageId = (int)($page['id'] ?? 0);
            $title = (string)($page['title'] ?? 'Страница');
            $url = sb_public_page_url($basePath, $siteId, $pageId);

            $childHtml = sb_public_render_auto_menu_level($pages, $pageId, $basePath, $siteId, $currentPageId, $level + 1);
            $hasChildren = $childHtml !== '';
            $isActive = $pageId === $currentPageId || sb_public_auto_menu_has_active_child($pages, $pageId, $currentPageId);

            $html .= '<div class="sb-public-menu__item' . ($hasChildren ? ' has-children' : '') . ($isActive ? ' is-active' : '') . '">';
            $html .= '<a class="sb-public-menu__link" href="' . sb_public_h($url) . '">';
            $html .= sb_public_h($title);

            if ($hasChildren) {
                $html .= ' <span class="sb-public-menu__arrow">▾</span>';
            }

            $html .= '</a>';

            if ($hasChildren) {
                $html .= $childHtml;
            }

            $html .= '</div>';
        }

        $html .= '</nav>';

        return $html;
    }
}

if (!function_exists('sb_public_render_auto_pages_menu')) {
    function sb_public_render_auto_pages_menu(array $pages, string $basePath, int $siteId, int $currentPageId = 0): string
    {
        return sb_public_render_auto_menu_level($pages, 0, $basePath, $siteId, $currentPageId, 0);
    }
}

/* =========================================================
   PAGE SECTIONS
   ========================================================= */

if (!function_exists('sb_public_normalize_public_section')) {
    function sb_public_normalize_public_section(array $section, int $siteId, int $pageId): array
    {
        $layout = sb_public_to_array($section['layout'] ?? []);
        $props = sb_public_to_array($section['props'] ?? []);

        $columns = sb_public_clamp_int($layout['columns'] ?? 1, 1, 4);
        $gap = sb_public_clamp_int($layout['gap'] ?? 24, 0, 120);

        $container = (string)($layout['container'] ?? 'default');

        if (!in_array($container, ['default', 'wide', 'full'], true)) {
            $container = 'default';
        }

        $layout['columns'] = $columns;
        $layout['gap'] = $gap;
        $layout['container'] = $container;

        return [
            'id' => (int)($section['id'] ?? 0),
            'siteId' => (int)($section['siteId'] ?? $section['site_id'] ?? $siteId),
            'pageId' => (int)($section['pageId'] ?? $section['page_id'] ?? $pageId),
            'title' => (string)($section['title'] ?? 'Секция'),
            'sort' => (int)($section['sort'] ?? 500),
            'layout' => $layout,
            'props' => $props,
        ];
    }
}

if (!function_exists('sb_public_page_sections_for_page')) {
    function sb_public_page_sections_for_page(array $vm, int $siteId, int $pageId): array
    {
        $sections = [];

        if (isset($vm['pageSections']) && is_array($vm['pageSections'])) {
            $sections = $vm['pageSections'];
        } elseif (isset($vm['sections']) && is_array($vm['sections'])) {
            $sections = $vm['sections'];
        }

        if (empty($sections) && function_exists('sb_read_page_sections')) {
            $sections = sb_read_page_sections();
        }

        if (empty($sections)) {
            $repoFile = dirname(__DIR__, 2) . '/lib/PageSectionRepository.php';

            if (!class_exists('PageSectionRepository') && file_exists($repoFile)) {
                require_once $repoFile;
            }

            if (class_exists('PageSectionRepository')) {
                $methods = [
                    'getByPage',
                    'listByPage',
                    'getList',
                    'getByPageId',
                    'getForPage',
                    'findByPage',
                    'forPage',
                    'pageSections',
                ];

                foreach ($methods as $method) {
                    if (!method_exists('PageSectionRepository', $method)) {
                        continue;
                    }

                    try {
                        $result = call_user_func(['PageSectionRepository', $method], $siteId, $pageId);

                        if (is_array($result)) {
                            $sections = isset($result['sections']) && is_array($result['sections'])
                                ? $result['sections']
                                : $result;
                            break;
                        }
                    } catch (Throwable $e) {
                    }

                    try {
                        $result = call_user_func(['PageSectionRepository', $method], $pageId);

                        if (is_array($result)) {
                            $sections = isset($result['sections']) && is_array($result['sections'])
                                ? $result['sections']
                                : $result;
                            break;
                        }
                    } catch (Throwable $e) {
                    }
                }
            }
        }

        if (!is_array($sections)) {
            $sections = [];
        }

        $sections = array_values(array_filter($sections, static function ($section) use ($pageId, $siteId) {
            $sectionPageId = (int)($section['pageId'] ?? $section['page_id'] ?? 0);
            $sectionSiteId = (int)($section['siteId'] ?? $section['site_id'] ?? 0);

            if ($sectionPageId > 0 && $sectionPageId !== $pageId) {
                return false;
            }

            if ($sectionSiteId > 0 && $sectionSiteId !== $siteId) {
                return false;
            }

            return true;
        }));

        $sections = array_map(static function ($section) use ($pageId, $siteId) {
            return sb_public_normalize_public_section($section, $siteId, $pageId);
        }, $sections);

        usort($sections, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

            if ($sortCmp !== 0) {
                return $sortCmp;
            }

            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $sections;
    }
}

if (!function_exists('sb_public_block_section_id')) {
    function sb_public_block_section_id(array $block): int
    {
        $props = sb_public_to_array($block['props'] ?? []);
        $placement = sb_public_to_array($props['_placement'] ?? []);

        $sectionId = (int)($block['sectionId'] ?? 0);

        if ($sectionId <= 0) {
            $sectionId = (int)($props['sectionId'] ?? 0);
        }

        if ($sectionId <= 0) {
            $sectionId = (int)($placement['sectionId'] ?? 0);
        }

        return $sectionId;
    }
}

if (!function_exists('sb_public_block_column')) {
    function sb_public_block_column(array $block): int
    {
        $props = sb_public_to_array($block['props'] ?? []);
        $placement = sb_public_to_array($props['_placement'] ?? []);

        $column = (int)($block['column'] ?? 0);

        if ($column <= 0) {
            $column = (int)($props['column'] ?? 0);
        }

        if ($column <= 0) {
            $column = (int)($placement['column'] ?? 0);
        }

        if ($column <= 0) {
            $column = 1;
        }

        return $column;
    }
}

if (!function_exists('sb_public_section_style')) {
    function sb_public_section_style(array $section): string
    {
        $props = sb_public_to_array($section['props'] ?? []);
        $styles = [];

        $backgroundColor = trim((string)($props['backgroundColor'] ?? ''));
        $backgroundImage = trim((string)($props['backgroundImage'] ?? ''));

        $paddingTop = sb_public_clamp_int($props['paddingTop'] ?? 0, 0, 400);
        $paddingBottom = sb_public_clamp_int($props['paddingBottom'] ?? 0, 0, 400);
        $minHeight = sb_public_clamp_int($props['minHeight'] ?? 0, 0, 2000);

        if ($backgroundColor !== '') {
            $styles[] = 'background-color:' . $backgroundColor;
        }

        if ($backgroundImage !== '') {
            $styles[] = 'background-image:url(\'' . str_replace("'", "\\'", $backgroundImage) . '\')';
            $styles[] = 'background-size:cover';
            $styles[] = 'background-position:center';
        }

        if ($paddingTop > 0) {
            $styles[] = 'padding-top:' . $paddingTop . 'px';
        }

        if ($paddingBottom > 0) {
            $styles[] = 'padding-bottom:' . $paddingBottom . 'px';
        }

        if ($minHeight > 0) {
            $styles[] = 'min-height:' . $minHeight . 'px';
        }

        return implode(';', $styles);
    }
}

if (!function_exists('sb_public_section_container_class')) {
    function sb_public_section_container_class(array $section): string
    {
        $layout = sb_public_to_array($section['layout'] ?? []);
        $container = (string)($layout['container'] ?? 'default');

        if (!in_array($container, ['default', 'wide', 'full'], true)) {
            $container = 'default';
        }

        return 'sb-section-container sb-section-container--' . $container;
    }
}

if (!function_exists('sb_public_group_blocks_by_section')) {
    function sb_public_group_blocks_by_section(array $pageBlocks, array $sections): array
    {
        $result = [];
        $firstSectionId = 0;

        foreach ($sections as $section) {
            $sectionId = (int)($section['id'] ?? 0);

            if ($sectionId <= 0) {
                continue;
            }

            if ($firstSectionId <= 0) {
                $firstSectionId = $sectionId;
            }

            $result[$sectionId] = [];
        }

        foreach ($pageBlocks as $block) {
            $sectionId = sb_public_block_section_id($block);

            if ($sectionId <= 0 || !isset($result[$sectionId])) {
                $sectionId = $firstSectionId;
            }

            if ($sectionId <= 0 || !isset($result[$sectionId])) {
                continue;
            }

            $result[$sectionId][] = $block;
        }

        return $result;
    }
}

if (!function_exists('sb_public_group_blocks_by_column')) {
    function sb_public_group_blocks_by_column(array $blocks, int $columns): array
    {
        $columns = max(1, min(4, $columns));

        $result = [];

        for ($i = 1; $i <= $columns; $i++) {
            $result[$i] = [];
        }

        foreach ($blocks as $block) {
            $column = sb_public_block_column($block);
            $column = max(1, min($columns, $column));

            $result[$column][] = $block;
        }

        return $result;
    }
}

if (!function_exists('sb_public_render_page_sections')) {
    function sb_public_render_page_sections(array $sections, array $pageBlocks, array $vm): string
    {
        if (empty($sections)) {
            return sb_public_render_blocks($pageBlocks, $vm);
        }

        $blocksBySection = sb_public_group_blocks_by_section($pageBlocks, $sections);

        $html = '<div class="sb-page-sections">';

        foreach ($sections as $section) {
            $sectionId = (int)($section['id'] ?? 0);

            if ($sectionId <= 0) {
                continue;
            }

            $sectionBlocks = $blocksBySection[$sectionId] ?? [];

            $layout = sb_public_to_array($section['layout'] ?? []);
            $columns = sb_public_clamp_int($layout['columns'] ?? 1, 1, 4);
            $gap = sb_public_clamp_int($layout['gap'] ?? 24, 0, 120);

            $sectionStyle = sb_public_section_style($section);
            $containerClass = sb_public_section_container_class($section);
            $columnBlocks = sb_public_group_blocks_by_column($sectionBlocks, $columns);

            $html .= '<section class="sb-page-section sb-page-section--columns-' . $columns . '"'
                . ($sectionStyle !== '' ? ' style="' . sb_public_h($sectionStyle) . '"' : '')
                . '>';

            $html .= '<div class="' . sb_public_h($containerClass) . '">';
            $html .= '<div class="sb-section-grid" style="--sb-section-columns:' . $columns . ';--sb-section-gap:' . $gap . 'px;">';

            for ($column = 1; $column <= $columns; $column++) {
                $html .= '<div class="sb-section-column sb-section-column--' . $column . '">';
                $html .= sb_public_render_blocks($columnBlocks[$column] ?? [], $vm);
                $html .= '</div>';
            }

            $html .= '</div>';
            $html .= '</div>';
            $html .= '</section>';
        }

        $html .= '</div>';

        return $html;
    }
}

if (!function_exists('sb_public_blocks_have_type')) {
    function sb_public_blocks_have_type(array $blocks, string $type): bool
    {
        foreach ($blocks as $block) {
            if ((string)($block['type'] ?? '') === $type) {
                return true;
            }
        }

        return false;
    }
}

/* =========================================================
   LAYOUT ZONES
   ========================================================= */

$headerBlocks = $layout['zones']['header'] ?? [];
$footerBlocks = $layout['zones']['footer'] ?? [];
$leftBlocks = $layout['zones']['left'] ?? [];
$rightBlocks = $layout['zones']['right'] ?? [];

if (!is_array($headerBlocks)) {
    $headerBlocks = [];
}

if (!is_array($footerBlocks)) {
    $footerBlocks = [];
}

if (!is_array($leftBlocks)) {
    $leftBlocks = [];
}

if (!is_array($rightBlocks)) {
    $rightBlocks = [];
}

$headerHtml = sb_public_render_blocks($headerBlocks, $vm);
$footerHtml = sb_public_render_blocks($footerBlocks, $vm);
$leftHtml = sb_public_render_blocks($leftBlocks, $vm);
$rightHtml = sb_public_render_blocks($rightBlocks, $vm);

$currentPageId = (int)($currentPage['id'] ?? 0);
$pageSections = $currentPageId > 0
    ? sb_public_page_sections_for_page($vm, $siteId, $currentPageId)
    : [];

$pageHtml = sb_public_render_page_sections($pageSections, $pageBlocks, $vm);
$menuHtml = sb_public_render_auto_pages_menu($pages, $basePath, $siteId, $currentPageId);

$pageHasDiskBlock = sb_public_blocks_have_type($pageBlocks, 'disk')
    || sb_public_blocks_have_type($headerBlocks, 'disk')
    || sb_public_blocks_have_type($footerBlocks, 'disk')
    || sb_public_blocks_have_type($leftBlocks, 'disk')
    || sb_public_blocks_have_type($rightBlocks, 'disk');

$leftContentHtml = (($vm['leftMode'] ?? '') === 'menu' && $menuHtml !== '') ? $menuHtml : $leftHtml;

if (($vm['leftMode'] ?? '') === 'menu' && !empty($vm['sectionNavHtml'])) {
    $leftContentHtml = (string)$vm['sectionNavHtml'];
}

global $APPLICATION;

if ($pageHasDiskBlock) {
    \CJSCore::Init([
        'viewer',
        'ui.viewer',
    ]);

    if (\Bitrix\Main\Loader::includeModule('disk')) {
        \Bitrix\Main\UI\Extension::load([
            'disk.viewer.document-item',
        ]);
    }
}
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">

    <?php $APPLICATION->ShowHead(); ?>

    <title><?= sb_public_h((string)($currentPage['title'] ?? $site['name'] ?? 'SiteBuilder')) ?></title>

    <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=7">

    <?php if ($pageHasDiskBlock): ?>
        <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=7">
    <?php endif; ?>

    <style>
        :root {
            --sb-accent: <?= sb_public_h((string)($vm['accent'] ?? '#2563eb')) ?>;
            --sb-container-width: <?= (int)($vm['containerWidth'] ?? 1360) ?>px;
            --sb-left-width: <?= (int)($vm['leftWidth'] ?? 260) ?>px;
            --sb-right-width: <?= (int)($vm['rightWidth'] ?? 260) ?>px;
        }
    </style>
</head>
<body>
<div class="sb-public-shell">
    <?php if (!empty($vm['showHeader'])): ?>
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
                <?php endif; ?>
            </div>
        </header>
    <?php endif; ?>

    <main class="sb-public-main">
        <div class="sb-container">
            <?php if (!empty($vm['breadcrumbsHtml'])): ?>
                <?= $vm['breadcrumbsHtml'] ?>
            <?php endif; ?>

            <div class="sb-layout <?= !empty($vm['showLeft']) ? 'sb-layout--left' : '' ?> <?= !empty($vm['showRight']) ? 'sb-layout--right' : '' ?>">
                <?php if (!empty($vm['showLeft'])): ?>
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

                <?php if (!empty($vm['showRight'])): ?>
                    <aside class="sb-sidebar sb-sidebar--right">
                        <div class="sb-box">
                            <?= $rightHtml !== '' ? $rightHtml : '<div class="sb-empty">Правая зона пуста</div>' ?>
                        </div>
                    </aside>
                <?php endif; ?>
            </div>
        </div>
    </main>

    <?php if (!empty($vm['showFooter'])): ?>
        <footer class="sb-public-footer">
            <div class="sb-container">
                <?= $footerHtml !== '' ? $footerHtml : '<div class="sb-footer-note">© ' . date('Y') . ' ' . sb_public_h((string)($site['name'] ?? 'SiteBuilder')) . '</div>' ?>
            </div>
        </footer>
    <?php endif; ?>
</div>

<script>
document.addEventListener('click', function (e) {
    var toggle = e.target.closest('[data-role="toggle"]');

    if (!toggle) {
        return;
    }

    var node = toggle.closest('.sb-tree-node');

    if (!node) {
        return;
    }

    var isOpen = node.classList.contains('is-open');

    node.classList.toggle('is-open', !isOpen);
    toggle.setAttribute('aria-expanded', !isOpen ? 'true' : 'false');
});
</script>

<?php if ($pageHasDiskBlock): ?>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=7"></script>
<?php endif; ?>

</body>
</html>

И проверь, что в конце public.css есть вот это:

.sb-page-sections {
    width: 100%;
    min-width: 0;
}

.sb-page-section {
    width: 100%;
    min-width: 0;
    box-sizing: border-box;
}

.sb-section-container {
    width: 100%;
    min-width: 0;
    margin: 0 auto;
    box-sizing: border-box;
}

.sb-section-container--default {
    max-width: var(--sb-container-width, 1360px);
}

.sb-section-container--wide {
    max-width: 1440px;
}

.sb-section-container--full {
    max-width: none;
}

.sb-page-section > .sb-section-container > .sb-section-grid {
    display: grid;
    grid-template-columns: repeat(var(--sb-section-columns, 1), minmax(0, 1fr));
    gap: var(--sb-section-gap, 24px);
    width: 100%;
    min-width: 0;
    align-items: start;
    box-sizing: border-box;
}

.sb-page-section > .sb-section-container > .sb-section-grid > .sb-section-column {
    min-width: 0;
    box-sizing: border-box;
}

@media (max-width: 900px) {
    .sb-page-section > .sb-section-container > .sb-section-grid {
        grid-template-columns: 1fr;
    }
}

После замены сделай Ctrl + F5 на публичной странице.