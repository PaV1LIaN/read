Отлично. Тогда идём правильно: разрезаем публичный рендер по файлам.

Сейчас сделаем нормальную структуру для публичной части:

/local/sitebuilder/
  public.php
  lib/
    public_render.php
  views/
    layout/
      public_page.php
    blocks/
      text.php
      heading.php
      button.php
      html.php
      spacer.php
  assets/
    public/
      public.css

Что это даст:

public.php станет коротким

рендер блоков уйдёт из него

каждый тип блока будет жить в своём файле

стили уйдут в отдельный CSS

дальше можно будет спокойно добавлять новые block types



---

1. Создай файл /local/sitebuilder/lib/public_render.php

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

        return $pages;
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
                return $page;
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

        $containerWidth = max(320, min(1920, (int)($settings['containerWidth'] ?? 1100)));
        $accent = (string)($settings['accent'] ?? '#2563eb');
        if (!preg_match('/^#[0-9a-fA-F]{6}$/', $accent)) {
            $accent = '#2563eb';
        }

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
        ];
    }
}


---

2. Создай файл /local/sitebuilder/views/blocks/text.php

<?php
$html = (string)($content['html'] ?? '');
?>
<section class="sb-block sb-block--text">
    <div class="sb-block__inner sb-text">
        <?= $html ?>
    </div>
</section>


---

3. Создай файл /local/sitebuilder/views/blocks/heading.php

<?php
$level = strtolower((string)($content['level'] ?? 'h2'));
if (!in_array($level, ['h1', 'h2', 'h3', 'h4', 'h5', 'h6'], true)) {
    $level = 'h2';
}
$text = sb_public_h((string)($content['text'] ?? ''));
$align = (string)($content['align'] ?? 'left');
if (!in_array($align, ['left', 'center', 'right'], true)) {
    $align = 'left';
}
?>
<section class="sb-block sb-block--heading">
    <<?= $level ?> class="sb-heading sb-heading--<?= sb_public_h($level) ?>" style="text-align:<?= sb_public_h($align) ?>;">
        <?= $text ?>
    </<?= $level ?>>
</section>


---

4. Создай файл /local/sitebuilder/views/blocks/button.php

<?php
$text = sb_public_h((string)($content['text'] ?? 'Кнопка'));
$href = sb_public_h((string)($content['href'] ?? '#'));
$target = sb_public_h((string)($content['target'] ?? '_self'));
$align = (string)($content['align'] ?? 'left');
if (!in_array($align, ['left', 'center', 'right'], true)) {
    $align = 'left';
}
?>
<section class="sb-block sb-block--button">
    <div class="sb-button-wrap" style="text-align:<?= sb_public_h($align) ?>;">
        <a class="sb-button" href="<?= $href ?>" target="<?= $target ?>">
            <?= $text ?>
        </a>
    </div>
</section>


---

5. Создай файл /local/sitebuilder/views/blocks/html.php

<?php
$html = (string)($content['html'] ?? '');
?>
<section class="sb-block sb-block--html">
    <div class="sb-block__inner">
        <?= $html ?>
    </div>
</section>


---

6. Создай файл /local/sitebuilder/views/blocks/spacer.php

<?php
$height = max(0, (int)($content['height'] ?? 30));
?>
<div class="sb-block sb-block--spacer" style="height:<?= $height ?>px;"></div>


---

7. Создай файл /local/sitebuilder/views/layout/public_page.php

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

8. Создай файл /local/sitebuilder/assets/public/public.css

* {
    box-sizing: border-box;
}

html,
body {
    margin: 0;
    padding: 0;
}

body {
    font-family: Arial, sans-serif;
    color: #1f2937;
    background: #f8fafc;
    line-height: 1.5;
}

.sb-public-shell {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
}

.sb-container {
    width: 100%;
    max-width: var(--sb-container-width);
    margin: 0 auto;
    padding: 0 20px;
}

.sb-public-header,
.sb-public-footer {
    background: #ffffff;
    border-bottom: 1px solid #e5e7eb;
}

.sb-public-footer {
    border-top: 1px solid #e5e7eb;
    border-bottom: 0;
    margin-top: auto;
}

.sb-public-header .sb-container,
.sb-public-footer .sb-container {
    padding-top: 20px;
    padding-bottom: 20px;
}

.sb-public-main .sb-container {
    padding-top: 24px;
    padding-bottom: 24px;
}

.sb-brand {
    font-size: 28px;
    font-weight: 700;
    color: var(--sb-accent);
    margin-bottom: 16px;
}

.sb-public-menu {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
}

.sb-public-menu__link {
    display: inline-block;
    text-decoration: none;
    color: var(--sb-accent);
    font-weight: 600;
    padding: 8px 12px;
    border-radius: 10px;
    transition: background .15s ease;
}

.sb-public-menu__link:hover {
    background: #eef2ff;
}

.sb-layout {
    display: grid;
    grid-template-columns: 1fr;
    gap: 20px;
    align-items: start;
}

.sb-layout.sb-layout--left {
    grid-template-columns: var(--sb-left-width) 1fr;
}

.sb-layout.sb-layout--right {
    grid-template-columns: 1fr var(--sb-right-width);
}

.sb-layout.sb-layout--left.sb-layout--right {
    grid-template-columns: var(--sb-left-width) 1fr var(--sb-right-width);
}

.sb-sidebar,
.sb-content {
    min-width: 0;
}

.sb-box {
    background: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    padding: 18px;
    box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
}

.sb-box--content {
    padding: 24px;
}

.sb-page-title {
    margin: 0 0 24px;
    font-size: 36px;
    line-height: 1.15;
}

.sb-block {
    margin: 0 0 18px;
}

.sb-block:last-child {
    margin-bottom: 0;
}

.sb-text {
    font-size: 16px;
    line-height: 1.7;
}

.sb-text p {
    margin: 0 0 14px;
}

.sb-text p:last-child {
    margin-bottom: 0;
}

.sb-heading {
    margin: 0 0 12px;
    line-height: 1.2;
}

.sb-heading--h1 {
    font-size: 40px;
}

.sb-heading--h2 {
    font-size: 32px;
}

.sb-heading--h3 {
    font-size: 26px;
}

.sb-heading--h4 {
    font-size: 22px;
}

.sb-heading--h5 {
    font-size: 18px;
}

.sb-heading--h6 {
    font-size: 16px;
}

.sb-button-wrap {
    margin: 0;
}

.sb-button {
    display: inline-block;
    padding: 12px 18px;
    border-radius: 12px;
    background: var(--sb-accent);
    color: #ffffff;
    text-decoration: none;
    font-weight: 700;
    transition: opacity .15s ease;
}

.sb-button:hover {
    opacity: 0.92;
}

.sb-empty {
    padding: 18px;
    border: 1px dashed #d1d5db;
    border-radius: 12px;
    color: #6b7280;
    background: #ffffff;
}

.sb-footer-note {
    color: #6b7280;
    font-size: 14px;
}

@media (max-width: 1000px) {
    .sb-layout,
    .sb-layout.sb-layout--left,
    .sb-layout.sb-layout--right,
    .sb-layout.sb-layout--left.sb-layout--right {
        grid-template-columns: 1fr;
    }

    .sb-page-title {
        font-size: 28px;
    }

    .sb-heading--h1 {
        font-size: 32px;
    }

    .sb-heading--h2 {
        font-size: 26px;
    }
}


---

9. Замени /local/sitebuilder/public.php

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

header('Content-Type: text/html; charset=UTF-8');

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$siteId = (int)($_GET['siteId'] ?? 0);
$pageId = isset($_GET['pageId']) ? (int)$_GET['pageId'] : null;

require_once __DIR__ . '/lib/public_render.php';

$vm = $siteId > 0 ? sb_public_build_view_model($siteId, $pageId, $basePath) : null;

if (!$vm) {
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>SiteBuilder / Public</title>
    </head>
    <body style="font-family:Arial,sans-serif;padding:24px;">
        <h1>Сайт не найден</h1>
        <p><a href="<?= sb_public_h($basePath) ?>/index.php">К списку сайтов</a></p>
    </body>
    </html>
    <?php
    exit;
}

include __DIR__ . '/views/layout/public_page.php';


---

10. Что проверить

Открой снова:

/local/sitebuilder/public.php?siteId=1

Проверь:

1. сайт стал визуально аккуратнее


2. есть нормальная шапка


3. меню сверху работает


4. блоки text, heading, button, html, spacer отображаются нормально


5. если в layout включить левую колонку — она отрисовывается


6. если leftMode = menu, слева показывается меню




---

11. Что мы сейчас сделали правильно

Мы не просто “подкрасили” public.php, а:

вынесли рендер в lib/public_render.php

сделали шаблоны блоков по файлам

вынесли CSS в assets/public/public.css

упростили public.php до точки входа


Это уже правильная структура.


---

12. Что делать дальше

Теперь у тебя уже есть рабочий backend и отдельно организованный публичный рендер.

Следующий правильный этап: точно так же разрезать административные страницы, начиная с editor.

То есть следующим сообщением я могу дать тебе уже правильную структуру для admin/frontend части:

общий API client js

общий admin css

вынос логики editor/menu/files/settings/layout из inline script в отдельные файлы


Напиши: давай разрежем admin-часть по файлам