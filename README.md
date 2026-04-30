Заменяй именно файл:

/local/sitebuilder/views/layout/public_page.php

на этот вариант. Он не конфликтует с lib/public_render.php и вызывает sb_public_render_blocks($blocks, $context) правильно.

<?php

global $APPLICATION, $USER;

if (!isset($vm) || !is_array($vm)) {
    $vm = [];
}

$basePath = (string)($vm['basePath'] ?? rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', dirname(__DIR__, 2)), '/'));
$site = is_array($vm['site'] ?? null) ? $vm['site'] : [];
$siteId = (int)($vm['siteId'] ?? ($vm['site_id'] ?? ($site['id'] ?? ($_GET['siteId'] ?? 0))));
$pages = is_array($vm['pages'] ?? null) ? $vm['pages'] : [];
$currentPage = is_array($vm['currentPage'] ?? null) ? $vm['currentPage'] : (is_array($vm['current_page'] ?? null) ? $vm['current_page'] : []);
$blocks = is_array($vm['blocks'] ?? null) ? $vm['blocks'] : [];
$layout = is_array($vm['layout'] ?? null) ? $vm['layout'] : [];

if (!function_exists('sb_public_view_h')) {
    function sb_public_view_h($value): string
    {
        return htmlspecialchars((string)$value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    }
}

if (!function_exists('sb_public_view_page_url')) {
    function sb_public_view_page_url(string $basePath, int $siteId, int $pageId = 0): string
    {
        $url = rtrim($basePath, '/') . '/public.php?siteId=' . $siteId;

        if ($pageId > 0) {
            $url .= '&pageId=' . $pageId;
        }

        return $url;
    }
}

if (!function_exists('sb_public_view_file_url')) {
    function sb_public_view_file_url(int $fileId): string
    {
        if ($fileId <= 0 || !class_exists('CFile')) {
            return '';
        }

        return (string)CFile::GetPath($fileId);
    }
}

if (!function_exists('sb_public_view_color')) {
    function sb_public_view_color(string $color, string $fallback): string
    {
        $color = trim($color);

        if (preg_match('/^#[0-9a-fA-F]{6}$/', $color) || preg_match('/^#[0-9a-fA-F]{3}$/', $color)) {
            return strtolower($color);
        }

        return $fallback;
    }
}

if (!function_exists('sb_public_view_background_size')) {
    function sb_public_view_background_size(string $mode): string
    {
        switch ($mode) {
            case 'contain':
                return 'contain';

            case 'auto':
                return 'auto';

            case 'stretch':
                return '100% 100%';

            case 'cover':
            default:
                return 'cover';
        }
    }
}

if (!function_exists('sb_public_view_background_position')) {
    function sb_public_view_background_position(string $position): string
    {
        $allowed = [
            'center center',
            'top center',
            'bottom center',
            'left center',
            'right center',
        ];

        return in_array($position, $allowed, true) ? $position : 'center center';
    }
}

if (!function_exists('sb_public_view_background_repeat')) {
    function sb_public_view_background_repeat(string $repeat): string
    {
        $allowed = [
            'no-repeat',
            'repeat',
            'repeat-x',
            'repeat-y',
        ];

        return in_array($repeat, $allowed, true) ? $repeat : 'no-repeat';
    }
}

if (!function_exists('sb_public_view_get_appearance')) {
    function sb_public_view_get_appearance(array $site): array
    {
        $settings = is_array($site['settings'] ?? null) ? $site['settings'] : [];

        $logoFileId = (int)($settings['logoFileId'] ?? 0);
        $backgroundFileId = (int)($settings['backgroundFileId'] ?? 0);

        $headerLogoMode = (string)($settings['headerLogoMode'] ?? 'image');
        if (!in_array($headerLogoMode, ['image', 'text', 'both'], true)) {
            $headerLogoMode = 'image';
        }

        return [
            'accent' => sb_public_view_color((string)($settings['accent'] ?? '#2563eb'), '#2563eb'),

            'logoFileId' => $logoFileId,
            'logoUrl' => sb_public_view_file_url($logoFileId),

            'backgroundFileId' => $backgroundFileId,
            'backgroundUrl' => sb_public_view_file_url($backgroundFileId),

            'backgroundColor' => sb_public_view_color((string)($settings['backgroundColor'] ?? '#f8fafc'), '#f8fafc'),
            'backgroundMode' => (string)($settings['backgroundMode'] ?? 'cover'),
            'backgroundPosition' => sb_public_view_background_position((string)($settings['backgroundPosition'] ?? 'center center')),
            'backgroundRepeat' => sb_public_view_background_repeat((string)($settings['backgroundRepeat'] ?? 'no-repeat')),

            'headerLogoMode' => $headerLogoMode,
        ];
    }
}

if (!function_exists('sb_public_view_root_style')) {
    function sb_public_view_root_style(array $appearance): string
    {
        $styles = [];

        $styles[] = '--sb-accent: ' . (string)($appearance['accent'] ?? '#2563eb');
        $styles[] = 'background-color: ' . (string)($appearance['backgroundColor'] ?? '#f8fafc');

        $backgroundUrl = (string)($appearance['backgroundUrl'] ?? '');

        if ($backgroundUrl !== '') {
            $styles[] = 'background-image: url("' . sb_public_view_h($backgroundUrl) . '")';
            $styles[] = 'background-size: ' . sb_public_view_background_size((string)($appearance['backgroundMode'] ?? 'cover'));
            $styles[] = 'background-position: ' . (string)($appearance['backgroundPosition'] ?? 'center center');
            $styles[] = 'background-repeat: ' . (string)($appearance['backgroundRepeat'] ?? 'no-repeat');
        }

        return implode('; ', $styles);
    }
}

if (!function_exists('sb_public_view_render_brand')) {
    function sb_public_view_render_brand(array $site, array $appearance): string
    {
        $siteName = (string)($site['name'] ?? 'Сайт');
        $logoUrl = (string)($appearance['logoUrl'] ?? '');
        $mode = (string)($appearance['headerLogoMode'] ?? 'image');

        if (!in_array($mode, ['image', 'text', 'both'], true)) {
            $mode = 'image';
        }

        $html = '';

        if (($mode === 'image' || $mode === 'both') && $logoUrl !== '') {
            $html .= '<span class="sb-public-brand__logo">';
            $html .= '<img src="' . sb_public_view_h($logoUrl) . '" alt="' . sb_public_view_h($siteName) . '">';
            $html .= '</span>';
        }

        if ($mode === 'text' || $mode === 'both' || $logoUrl === '') {
            $html .= '<span class="sb-public-brand__text">' . sb_public_view_h($siteName) . '</span>';
        }

        return $html;
    }
}

if (!function_exists('sb_public_view_page_visible')) {
    function sb_public_view_page_visible(array $page, int $currentPageId = 0): bool
    {
        $pageId = (int)($page['id'] ?? 0);
        $status = (string)($page['status'] ?? 'published');

        if ($pageId === $currentPageId) {
            return true;
        }

        return $status === 'published';
    }
}

if (!function_exists('sb_public_view_menu_children')) {
    function sb_public_view_menu_children(array $pages, int $parentId, int $currentPageId = 0): array
    {
        $items = [];

        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) !== $parentId) {
                continue;
            }

            if (!sb_public_view_page_visible($page, $currentPageId)) {
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

if (!function_exists('sb_public_view_has_active_child')) {
    function sb_public_view_has_active_child(array $pages, int $pageId, int $currentPageId): bool
    {
        foreach ($pages as $page) {
            if ((int)($page['parentId'] ?? 0) !== $pageId) {
                continue;
            }

            $childId = (int)($page['id'] ?? 0);

            if ($childId === $currentPageId) {
                return true;
            }

            if (sb_public_view_has_active_child($pages, $childId, $currentPageId)) {
                return true;
            }
        }

        return false;
    }
}

if (!function_exists('sb_public_view_render_menu_level')) {
    function sb_public_view_render_menu_level(array $pages, int $parentId, string $basePath, int $siteId, int $currentPageId, int $level = 0): string
    {
        $children = sb_public_view_menu_children($pages, $parentId, $currentPageId);

        if (empty($children)) {
            return '';
        }

        $class = $level === 0 ? 'sb-public-menu' : 'sb-public-menu__dropdown';
        $html = '<nav class="' . $class . '">';

        foreach ($children as $page) {
            $pageId = (int)($page['id'] ?? 0);
            $title = (string)($page['title'] ?? 'Страница');
            $url = sb_public_view_page_url($basePath, $siteId, $pageId);

            $childHtml = sb_public_view_render_menu_level($pages, $pageId, $basePath, $siteId, $currentPageId, $level + 1);
            $hasChildren = $childHtml !== '';
            $isActive = $pageId === $currentPageId || sb_public_view_has_active_child($pages, $pageId, $currentPageId);

            $html .= '<div class="sb-public-menu__item' . ($hasChildren ? ' has-children' : '') . ($isActive ? ' is-active' : '') . '">';
            $html .= '<a class="sb-public-menu__link" href="' . sb_public_view_h($url) . '">';
            $html .= sb_public_view_h($title);

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

if (!function_exists('sb_public_view_render_menu')) {
    function sb_public_view_render_menu(array $pages, string $basePath, int $siteId, int $currentPageId): string
    {
        return sb_public_view_render_menu_level($pages, 0, $basePath, $siteId, $currentPageId, 0);
    }
}

if (!function_exists('sb_public_view_get_zone_blocks')) {
    function sb_public_view_get_zone_blocks(array $layout, string $zone): array
    {
        if (isset($layout['zones']) && is_array($layout['zones']) && isset($layout['zones'][$zone]) && is_array($layout['zones'][$zone])) {
            return $layout['zones'][$zone];
        }

        if (isset($layout[$zone]) && is_array($layout[$zone])) {
            return $layout[$zone];
        }

        return [];
    }
}

if (!function_exists('sb_public_view_render_blocks')) {
    function sb_public_view_render_blocks(array $blocks, array $context): string
    {
        if (function_exists('sb_public_render_blocks')) {
            return (string)sb_public_render_blocks($blocks, $context);
        }

        if (empty($blocks)) {
            return '';
        }

        $html = '';

        foreach ($blocks as $block) {
            $type = (string)($block['type'] ?? 'text');
            $content = $block['content'] ?? [];

            if (is_string($content)) {
                $decoded = json_decode($content, true);
                $content = is_array($decoded) ? $decoded : [];
            }

            if (!is_array($content)) {
                $content = [];
            }

            if ($type === 'heading') {
                $html .= '<section class="sb-public-block"><h2>' . sb_public_view_h((string)($content['text'] ?? '')) . '</h2></section>';
            } elseif ($type === 'text') {
                $html .= '<section class="sb-public-block"><div>' . nl2br(sb_public_view_h((string)($content['text'] ?? ''))) . '</div></section>';
            } elseif ($type === 'html') {
                $html .= '<section class="sb-public-block">' . (string)($content['html'] ?? '') . '</section>';
            }
        }

        return $html;
    }
}

if (!function_exists('sb_public_view_render_layout_zone')) {
    function sb_public_view_render_layout_zone(array $layout, string $zone, array $context): string
    {
        $blocks = sb_public_view_get_zone_blocks($layout, $zone);

        if (empty($blocks)) {
            return '';
        }

        return sb_public_view_render_blocks($blocks, $context);
    }
}

$settings = is_array($site['settings'] ?? null) ? $site['settings'] : [];
$siteLayout = is_array($site['layout'] ?? null) ? $site['layout'] : [];
$layoutSettings = is_array($layout['settings'] ?? null) ? $layout['settings'] : $siteLayout;

$currentPageId = (int)($currentPage['id'] ?? 0);

$appearance = sb_public_view_get_appearance($site);
$rootStyle = sb_public_view_root_style($appearance);

$containerWidth = (int)($settings['containerWidth'] ?? 1100);
if ($containerWidth <= 0) {
    $containerWidth = 1100;
}

$showHeader = array_key_exists('showHeader', $layoutSettings) ? (bool)$layoutSettings['showHeader'] : true;
$showFooter = array_key_exists('showFooter', $layoutSettings) ? (bool)$layoutSettings['showFooter'] : true;
$showLeft = array_key_exists('showLeft', $layoutSettings) ? (bool)$layoutSettings['showLeft'] : false;
$showRight = array_key_exists('showRight', $layoutSettings) ? (bool)$layoutSettings['showRight'] : false;

$leftWidth = (int)($layoutSettings['leftWidth'] ?? 260);
$rightWidth = (int)($layoutSettings['rightWidth'] ?? 260);

if ($leftWidth <= 0) {
    $leftWidth = 260;
}

if ($rightWidth <= 0) {
    $rightWidth = 260;
}

$siteName = (string)($site['name'] ?? 'Сайт');
$pageTitle = (string)($currentPage['title'] ?? $siteName);

$renderContext = [
    'siteId' => $siteId,
    'site_id' => $siteId,

    'pageId' => $currentPageId,
    'page_id' => $currentPageId,

    'basePath' => $basePath,
    'base_path' => $basePath,

    'site' => $site,
    'pages' => $pages,
    'currentPage' => $currentPage,
    'current_page' => $currentPage,
    'layout' => $layout,
];

$menuHtml = sb_public_view_render_menu($pages, $basePath, $siteId, $currentPageId);

$headerHtml = sb_public_view_render_layout_zone($layout, 'header', $renderContext);
$leftHtml = sb_public_view_render_layout_zone($layout, 'left', $renderContext);
$rightHtml = sb_public_view_render_layout_zone($layout, 'right', $renderContext);
$footerHtml = sb_public_view_render_layout_zone($layout, 'footer', $renderContext);

$hasLeft = $showLeft && $leftHtml !== '';
$hasRight = $showRight && $rightHtml !== '';

$layoutColumns = [];

if ($hasLeft) {
    $layoutColumns[] = $leftWidth . 'px';
}

$layoutColumns[] = 'minmax(0, 1fr)';

if ($hasRight) {
    $layoutColumns[] = $rightWidth . 'px';
}

$mainGridStyle = '';
if ($hasLeft || $hasRight) {
    $mainGridStyle = 'grid-template-columns: ' . implode(' ', $layoutColumns);
}

$diskCssPath = dirname(__DIR__, 2) . '/components/disk/assets/disk.css';
$diskJsPath = dirname(__DIR__, 2) . '/components/disk/assets/disk.js';

$hasDiskCss = file_exists($diskCssPath);
$hasDiskJs = file_exists($diskJsPath);

?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title><?= sb_public_view_h($pageTitle) ?></title>

    <?php if (is_object($APPLICATION)): ?>
        <?php $APPLICATION->ShowHead(); ?>
    <?php endif; ?>

    <link rel="stylesheet" href="<?= sb_public_view_h($basePath) ?>/assets/public/public.css?v=5">

    <?php if ($hasDiskCss): ?>
        <link rel="stylesheet" href="<?= sb_public_view_h($basePath) ?>/components/disk/assets/disk.css?v=1">
    <?php endif; ?>

    <style>
        .sb-public-root {
            min-height: 100vh;
            background-attachment: fixed;
        }

        .sb-public-container {
            width: min(100% - 32px, var(--sb-container-width, 1100px));
            margin-left: auto;
            margin-right: auto;
        }

        .sb-public-header {
            background: rgba(255, 255, 255, .92);
            border-bottom: 1px solid rgba(229, 231, 235, .85);
            backdrop-filter: blur(10px);
        }

        .sb-public-header__inner {
            min-height: 74px;
            display: flex;
            align-items: center;
            justify-content: space-between;
            gap: 24px;
        }

        .sb-public-brand {
            display: inline-flex;
            align-items: center;
            gap: 10px;
            min-width: 0;
            color: #111827;
            text-decoration: none;
            font-weight: 800;
            font-size: 18px;
        }

        .sb-public-brand:hover {
            color: var(--sb-accent, #2563eb);
        }

        .sb-public-brand__logo {
            width: 42px;
            height: 42px;
            border-radius: 12px;
            overflow: hidden;
            border: 1px solid #e5e7eb;
            background: #fff;
            display: inline-flex;
            align-items: center;
            justify-content: center;
            flex: 0 0 auto;
        }

        .sb-public-brand__logo img {
            max-width: 100%;
            max-height: 100%;
            object-fit: contain;
            display: block;
        }

        .sb-public-brand__text {
            min-width: 0;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }

        .sb-public-menu {
            display: flex;
            align-items: center;
            flex-wrap: wrap;
            gap: 6px;
        }

        .sb-public-menu__item {
            position: relative;
        }

        .sb-public-menu__link {
            display: inline-flex;
            align-items: center;
            gap: 4px;
            min-height: 36px;
            padding: 0 12px;
            border-radius: 10px;
            color: #111827;
            text-decoration: none;
            font-size: 14px;
            font-weight: 600;
        }

        .sb-public-menu__link:hover,
        .sb-public-menu__item.is-active > .sb-public-menu__link {
            background: rgba(37, 99, 235, 0.08);
            color: var(--sb-accent, #2563eb);
        }

        .sb-public-menu__dropdown {
            position: absolute;
            left: 0;
            top: calc(100% + 6px);
            z-index: 50;
            min-width: 220px;
            display: none;
            flex-direction: column;
            gap: 2px;
            padding: 8px;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            background: #fff;
            box-shadow: 0 18px 45px rgba(15, 23, 42, .14);
        }

        .sb-public-menu__item:hover > .sb-public-menu__dropdown {
            display: flex;
        }

        .sb-public-menu__dropdown .sb-public-menu__dropdown {
            left: calc(100% + 8px);
            top: 0;
        }

        .sb-public-main {
            padding: 32px 0;
        }

        .sb-public-layout {
            display: grid;
            gap: 24px;
            align-items: start;
        }

        .sb-public-page-card,
        .sb-public-sidebar,
        .sb-public-footer__inner {
            background: rgba(255, 255, 255, .92);
            border: 1px solid rgba(229, 231, 235, .9);
            border-radius: 18px;
            box-shadow: 0 10px 30px rgba(15, 23, 42, .05);
        }

        .sb-public-page-card {
            padding: 28px;
            min-height: 360px;
        }

        .sb-public-sidebar {
            padding: 18px;
        }

        .sb-public-page-title {
            margin: 0 0 22px;
            font-size: 34px;
            line-height: 1.15;
            font-weight: 800;
            color: #111827;
        }

        .sb-public-footer {
            padding: 0 0 32px;
        }

        .sb-public-footer__inner {
            padding: 18px 22px;
            color: #6b7280;
        }

        .sb-public-empty {
            display: flex;
            flex-direction: column;
            gap: 6px;
            padding: 28px;
            border: 1px dashed #d1d5db;
            border-radius: 16px;
            color: #6b7280;
            background: rgba(255, 255, 255, .7);
        }

        .sb-public-empty strong {
            color: #111827;
        }

        @media (max-width: 900px) {
            .sb-public-root {
                background-attachment: scroll;
            }

            .sb-public-header__inner {
                align-items: flex-start;
                flex-direction: column;
                padding: 14px 0;
                gap: 12px;
            }

            .sb-public-menu {
                width: 100%;
                flex-direction: column;
                align-items: stretch;
            }

            .sb-public-menu__link {
                width: 100%;
                justify-content: space-between;
            }

            .sb-public-menu__dropdown,
            .sb-public-menu__dropdown .sb-public-menu__dropdown {
                position: static;
                display: flex;
                box-shadow: none;
                margin-left: 12px;
                min-width: 0;
            }

            .sb-public-layout {
                grid-template-columns: 1fr !important;
            }

            .sb-public-page-card {
                padding: 20px;
            }

            .sb-public-page-title {
                font-size: 28px;
            }
        }
    </style>
</head>
<body class="sb-public-body">
<div class="sb-public-root" style="<?= sb_public_view_h($rootStyle) ?>; --sb-container-width: <?= (int)$containerWidth ?>px;">
    <?php if ($showHeader): ?>
        <header class="sb-public-header">
            <div class="sb-public-container">
                <div class="sb-public-header__inner">
                    <a class="sb-public-brand" href="<?= sb_public_view_h(sb_public_view_page_url($basePath, $siteId, 0)) ?>">
                        <?= sb_public_view_render_brand($site, $appearance) ?>
                    </a>

                    <?php if ($menuHtml !== ''): ?>
                        <?= $menuHtml ?>
                    <?php endif; ?>
                </div>

                <?php if ($headerHtml !== ''): ?>
                    <div class="sb-public-header__layout">
                        <?= $headerHtml ?>
                    </div>
                <?php endif; ?>
            </div>
        </header>
    <?php endif; ?>

    <main class="sb-public-main">
        <div class="sb-public-container">
            <div class="sb-public-layout<?= $hasLeft ? ' has-left' : '' ?><?= $hasRight ? ' has-right' : '' ?>" style="<?= sb_public_view_h($mainGridStyle) ?>">
                <?php if ($hasLeft): ?>
                    <aside class="sb-public-sidebar sb-public-sidebar--left">
                        <?= $leftHtml ?>
                    </aside>
                <?php endif; ?>

                <article class="sb-public-content">
                    <div class="sb-public-page-card">
                        <?php if ($currentPageId > 0): ?>
                            <h1 class="sb-public-page-title"><?= sb_public_view_h($pageTitle) ?></h1>

                            <?php if (!empty($blocks)): ?>
                                <?= sb_public_view_render_blocks($blocks, $renderContext) ?>
                            <?php else: ?>
                                <div class="sb-public-empty">
                                    <strong>На странице пока нет блоков</strong>
                                    <span>Добавь блоки в редакторе сайта.</span>
                                </div>
                            <?php endif; ?>
                        <?php else: ?>
                            <div class="sb-public-empty">
                                <strong>Страница не найдена</strong>
                                <span>Проверь, что у сайта есть опубликованная домашняя страница.</span>
                            </div>
                        <?php endif; ?>
                    </div>
                </article>

                <?php if ($hasRight): ?>
                    <aside class="sb-public-sidebar sb-public-sidebar--right">
                        <?= $rightHtml ?>
                    </aside>
                <?php endif; ?>
            </div>
        </div>
    </main>

    <?php if ($showFooter): ?>
        <footer class="sb-public-footer">
            <div class="sb-public-container">
                <div class="sb-public-footer__inner">
                    <?php if ($footerHtml !== ''): ?>
                        <?= $footerHtml ?>
                    <?php else: ?>
                        <?= sb_public_view_h($siteName) ?>
                    <?php endif; ?>
                </div>
            </div>
        </footer>
    <?php endif; ?>
</div>

<script src="/bitrix/js/main/core/core.js"></script>

<?php if ($hasDiskJs): ?>
    <script src="<?= sb_public_view_h($basePath) ?>/components/disk/assets/disk.js?v=1"></script>
<?php endif; ?>
</body>
</html>

После замены обнови публичную страницу через Ctrl + F5.