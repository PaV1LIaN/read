Заменяй полностью файл:

/local/sitebuilder/public_page.php

на этот:

<?php

global $APPLICATION, $USER;

if (!isset($vm) || !is_array($vm)) {
    $vm = [];
}

$basePath = (string)($vm['basePath'] ?? rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/'));
$site = is_array($vm['site'] ?? null) ? $vm['site'] : [];
$siteId = (int)($vm['siteId'] ?? ($site['id'] ?? ($_GET['siteId'] ?? 0)));
$pages = is_array($vm['pages'] ?? null) ? $vm['pages'] : [];
$currentPage = is_array($vm['currentPage'] ?? null) ? $vm['currentPage'] : [];
$blocks = is_array($vm['blocks'] ?? null) ? $vm['blocks'] : [];
$layout = is_array($vm['layout'] ?? null) ? $vm['layout'] : [];

if (!function_exists('sb_public_h')) {
    function sb_public_h($value): string
    {
        return htmlspecialchars((string)$value, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
    }
}

if (!function_exists('sb_public_page_url')) {
    function sb_public_page_url(string $basePath, int $siteId, int $pageId = 0): string
    {
        $url = rtrim($basePath, '/') . '/public.php?siteId=' . $siteId;

        if ($pageId > 0) {
            $url .= '&pageId=' . $pageId;
        }

        return $url;
    }
}

if (!function_exists('sb_public_get_file_url')) {
    function sb_public_get_file_url(int $fileId): string
    {
        if ($fileId <= 0) {
            return '';
        }

        if (!class_exists('CFile')) {
            return '';
        }

        return (string)CFile::GetPath($fileId);
    }
}

if (!function_exists('sb_public_normalize_background_size')) {
    function sb_public_normalize_background_size(string $mode): string
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

if (!function_exists('sb_public_get_appearance')) {
    function sb_public_get_appearance(array $site): array
    {
        $settings = is_array($site['settings'] ?? null) ? $site['settings'] : [];

        $logoFileId = (int)($settings['logoFileId'] ?? 0);
        $backgroundFileId = (int)($settings['backgroundFileId'] ?? 0);

        return [
            'accent' => (string)($settings['accent'] ?? '#2563eb'),

            'logoFileId' => $logoFileId,
            'logoUrl' => sb_public_get_file_url($logoFileId),

            'backgroundFileId' => $backgroundFileId,
            'backgroundUrl' => sb_public_get_file_url($backgroundFileId),

            'backgroundColor' => (string)($settings['backgroundColor'] ?? '#f8fafc'),
            'backgroundMode' => (string)($settings['backgroundMode'] ?? 'cover'),
            'backgroundPosition' => (string)($settings['backgroundPosition'] ?? 'center center'),
            'backgroundRepeat' => (string)($settings['backgroundRepeat'] ?? 'no-repeat'),

            'headerLogoMode' => (string)($settings['headerLogoMode'] ?? 'image'),
        ];
    }
}

if (!function_exists('sb_public_appearance_style')) {
    function sb_public_appearance_style(array $appearance): string
    {
        $styles = [];

        $accent = (string)($appearance['accent'] ?? '#2563eb');
        $backgroundColor = (string)($appearance['backgroundColor'] ?? '#f8fafc');
        $backgroundUrl = (string)($appearance['backgroundUrl'] ?? '');

        $styles[] = '--sb-accent: ' . $accent;
        $styles[] = 'background-color: ' . $backgroundColor;

        if ($backgroundUrl !== '') {
            $styles[] = 'background-image: url("' . sb_public_h($backgroundUrl) . '")';
            $styles[] = 'background-size: ' . sb_public_normalize_background_size((string)($appearance['backgroundMode'] ?? 'cover'));
            $styles[] = 'background-position: ' . (string)($appearance['backgroundPosition'] ?? 'center center');
            $styles[] = 'background-repeat: ' . (string)($appearance['backgroundRepeat'] ?? 'no-repeat');
        }

        return implode('; ', $styles);
    }
}

if (!function_exists('sb_public_render_brand')) {
    function sb_public_render_brand(array $site, array $appearance): string
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
            $html .= '<img src="' . sb_public_h($logoUrl) . '" alt="' . sb_public_h($siteName) . '">';
            $html .= '</span>';
        }

        if ($mode === 'text' || $mode === 'both' || $logoUrl === '') {
            $html .= '<span class="sb-public-brand__text">' . sb_public_h($siteName) . '</span>';
        }

        return $html;
    }
}

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

if (!function_exists('sb_public_get_block_array')) {
    function sb_public_get_block_array(array $block, string $key): array
    {
        $value = $block[$key] ?? [];

        if (is_array($value)) {
            return $value;
        }

        if (is_string($value) && $value !== '') {
            $decoded = json_decode($value, true);
            return is_array($decoded) ? $decoded : [];
        }

        return [];
    }
}

if (!function_exists('sb_public_render_block')) {
    function sb_public_render_block(array $block, int $siteId, int $pageId, string $basePath): string
    {
        $type = (string)($block['type'] ?? 'text');
        $content = sb_public_get_block_array($block, 'content');
        $props = sb_public_get_block_array($block, 'props');

        if ($type === 'heading') {
            $text = (string)($content['text'] ?? '');

            if ($text === '') {
                return '';
            }

            return '
                <section class="sb-public-block sb-public-block--heading">
                    <h2 class="sb-public-heading">' . sb_public_h($text) . '</h2>
                </section>
            ';
        }

        if ($type === 'text') {
            $text = (string)($content['text'] ?? '');

            if ($text === '') {
                return '';
            }

            return '
                <section class="sb-public-block sb-public-block--text">
                    <div class="sb-public-text">' . nl2br(sb_public_h($text)) . '</div>
                </section>
            ';
        }

        if ($type === 'button') {
            $label = (string)($content['label'] ?? 'Кнопка');
            $href = (string)($content['href'] ?? '#');
            $target = (string)($content['target'] ?? '_self');

            if (!in_array($target, ['_self', '_blank'], true)) {
                $target = '_self';
            }

            return '
                <section class="sb-public-block sb-public-block--button">
                    <a class="sb-public-button" href="' . sb_public_h($href) . '" target="' . sb_public_h($target) . '">
                        ' . sb_public_h($label) . '
                    </a>
                </section>
            ';
        }

        if ($type === 'html') {
            $html = (string)($content['html'] ?? '');

            if ($html === '') {
                return '';
            }

            return '
                <section class="sb-public-block sb-public-block--html">
                    ' . $html . '
                </section>
            ';
        }

        if ($type === 'disk') {
            return sb_public_render_disk_block($block, $siteId, $pageId, $basePath, $props);
        }

        return '
            <section class="sb-public-block sb-public-block--unknown">
                <pre>' . sb_public_h(json_encode($content, JSON_PRETTY_PRINT | JSON_UNESCAPED_UNICODE)) . '</pre>
            </section>
        ';
    }
}

if (!function_exists('sb_public_render_disk_block')) {
    function sb_public_render_disk_block(array $block, int $siteId, int $pageId, string $basePath, array $props): string
    {
        $blockId = (int)($block['id'] ?? 0);
        $title = (string)($props['title'] ?? 'Файлы');

        $sessid = '';
        if (function_exists('bitrix_sessid')) {
            $sessid = (string)bitrix_sessid();
        }

        return '
            <section class="sb-public-block sb-public-block--disk">
                <div class="sb-public-disk-head">
                    <h2 class="sb-public-block-title">' . sb_public_h($title) . '</h2>
                </div>

                <div
                    class="sb-disk"
                    data-site-id="' . (int)$siteId . '"
                    data-page-id="' . (int)$pageId . '"
                    data-block-id="' . (int)$blockId . '"
                    data-sessid="' . sb_public_h($sessid) . '"
                >
                    <div class="sb-public-disk-loading">Загрузка файлов...</div>
                </div>
            </section>
        ';
    }
}

if (!function_exists('sb_public_render_blocks')) {
    function sb_public_render_blocks(array $blocks, int $siteId, int $pageId, string $basePath): string
    {
        if (empty($blocks)) {
            return '
                <div class="sb-public-empty">
                    <strong>На странице пока нет блоков</strong>
                    <span>Добавь блоки в редакторе сайта.</span>
                </div>
            ';
        }

        usort($blocks, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

            if ($sortCmp !== 0) {
                return $sortCmp;
            }

            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        $html = '';

        foreach ($blocks as $block) {
            $html .= sb_public_render_block($block, $siteId, $pageId, $basePath);
        }

        return $html;
    }
}

if (!function_exists('sb_public_render_layout_blocks')) {
    function sb_public_render_layout_blocks(array $layout, string $zone, int $siteId, int $pageId, string $basePath): string
    {
        $zones = is_array($layout['zones'] ?? null) ? $layout['zones'] : [];
        $blocks = is_array($zones[$zone] ?? null) ? $zones[$zone] : [];

        if (empty($blocks)) {
            return '';
        }

        return sb_public_render_blocks($blocks, $siteId, $pageId, $basePath);
    }
}

$settings = is_array($site['settings'] ?? null) ? $site['settings'] : [];
$siteLayout = is_array($site['layout'] ?? null) ? $site['layout'] : [];
$layoutSettings = is_array($layout['settings'] ?? null) ? $layout['settings'] : $siteLayout;

$currentPageId = (int)($currentPage['id'] ?? 0);

$appearance = sb_public_get_appearance($site);
$publicRootStyle = sb_public_appearance_style($appearance);

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
$menuHtml = sb_public_render_auto_pages_menu($pages, $basePath, $siteId, $currentPageId);

$leftHtml = sb_public_render_layout_blocks($layout, 'left', $siteId, $currentPageId, $basePath);
$rightHtml = sb_public_render_layout_blocks($layout, 'right', $siteId, $currentPageId, $basePath);
$footerHtml = sb_public_render_layout_blocks($layout, 'footer', $siteId, $currentPageId, $basePath);
$headerHtml = sb_public_render_layout_blocks($layout, 'header', $siteId, $currentPageId, $basePath);

$hasLeft = $showLeft && $leftHtml !== '';
$hasRight = $showRight && $rightHtml !== '';

$mainGridStyle = [];
if ($hasLeft || $hasRight) {
    $columns = [];

    if ($hasLeft) {
        $columns[] = $leftWidth . 'px';
    }

    $columns[] = 'minmax(0, 1fr)';

    if ($hasRight) {
        $columns[] = $rightWidth . 'px';
    }

    $mainGridStyle[] = 'grid-template-columns: ' . implode(' ', $columns);
}

$mainGridStyleAttr = implode('; ', $mainGridStyle);

$diskCssPath = __DIR__ . '/components/disk/assets/disk.css';
$diskJsPath = __DIR__ . '/components/disk/assets/disk.js';

$hasDiskCss = file_exists($diskCssPath);
$hasDiskJs = file_exists($diskJsPath);

?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title><?= sb_public_h($pageTitle) ?></title>
    <?php if (is_object($APPLICATION)): ?>
        <?php $APPLICATION->ShowHead(); ?>
    <?php endif; ?>

    <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=3">

    <?php if ($hasDiskCss): ?>
        <link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/assets/disk.css?v=1">
    <?php endif; ?>
</head>
<body class="sb-public-body">
<div class="sb-public-root" style="<?= sb_public_h($publicRootStyle) ?>">
    <?php if ($showHeader): ?>
        <header class="sb-public-header">
            <div class="sb-public-container" style="max-width: <?= (int)$containerWidth ?>px">
                <div class="sb-public-header__inner">
                    <a class="sb-public-brand" href="<?= sb_public_h(sb_public_page_url($basePath, $siteId, 0)) ?>">
                        <?= sb_public_render_brand($site, $appearance) ?>
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
        <div class="sb-public-container" style="max-width: <?= (int)$containerWidth ?>px">
            <div class="sb-public-layout<?= $hasLeft ? ' has-left' : '' ?><?= $hasRight ? ' has-right' : '' ?>" style="<?= sb_public_h($mainGridStyleAttr) ?>">
                <?php if ($hasLeft): ?>
                    <aside class="sb-public-sidebar sb-public-sidebar--left">
                        <?= $leftHtml ?>
                    </aside>
                <?php endif; ?>

                <article class="sb-public-content">
                    <?php if ($currentPageId > 0): ?>
                        <div class="sb-public-page-card">
                            <h1 class="sb-public-page-title"><?= sb_public_h($pageTitle) ?></h1>
                            <?= sb_public_render_blocks($blocks, $siteId, $currentPageId, $basePath) ?>
                        </div>
                    <?php else: ?>
                        <div class="sb-public-page-card">
                            <div class="sb-public-empty">
                                <strong>Страница не найдена</strong>
                                <span>Проверь, что у сайта есть опубликованная домашняя страница.</span>
                            </div>
                        </div>
                    <?php endif; ?>
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
            <div class="sb-public-container" style="max-width: <?= (int)$containerWidth ?>px">
                <?php if ($footerHtml !== ''): ?>
                    <?= $footerHtml ?>
                <?php else: ?>
                    <div class="sb-public-footer__default">
                        <?= sb_public_h($siteName) ?>
                    </div>
                <?php endif; ?>
            </div>
        </footer>
    <?php endif; ?>
</div>

<script src="/bitrix/js/main/core/core.js"></script>

<?php if ($hasDiskJs): ?>
    <script src="<?= sb_public_h($basePath) ?>/components/disk/assets/disk.js?v=1"></script>
<?php endif; ?>
</body>
</html>