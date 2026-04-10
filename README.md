Отлично. Тогда следующий логичный шаг — сделать public.php, чтобы уже можно было открывать сайт не как админку, а как публичную страницу.

Сейчас сделаем рабочую версию, которая:

грузит сайт по siteId

берёт layout

рендерит header / footer / left / right

рендерит домашнюю страницу сайта

если homePageId не задан — берёт первую страницу сайта

выводит page blocks

поддерживает базовые типы блоков:

text

heading

button

html

spacer



Это уже даст тебе живой просмотр результата.


---

/local/sitebuilder/public.php

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

global $APPLICATION;

header('Content-Type: text/html; charset=UTF-8');

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$siteId = (int)($_GET['siteId'] ?? 0);

require_once __DIR__ . '/lib/json.php';
require_once __DIR__ . '/lib/helpers.php';

function sb_public_h(string $s): string
{
    return htmlspecialchars($s, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8');
}

function sb_public_find_site(int $siteId): ?array
{
    foreach (sb_read_sites() as $site) {
        if ((int)($site['id'] ?? 0) === $siteId) {
            return $site;
        }
    }
    return null;
}

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

    return $blocks;
}

function sb_public_layout_for_site(int $siteId): array
{
    foreach (sb_read_layouts() as $layout) {
        if ((int)($layout['siteId'] ?? 0) === $siteId) {
            return sb_normalize_layout_record($layout);
        }
    }

    return sb_layout_default_record($siteId);
}

function sb_public_render_block(array $block): string
{
    $type = (string)($block['type'] ?? 'text');
    $content = isset($block['content']) && is_array($block['content']) ? $block['content'] : [];

    switch ($type) {
        case 'heading':
            $level = strtolower((string)($content['level'] ?? 'h2'));
            if (!in_array($level, ['h1', 'h2', 'h3', 'h4', 'h5', 'h6'], true)) {
                $level = 'h2';
            }
            $text = sb_public_h((string)($content['text'] ?? ''));
            $align = sb_public_h((string)($content['align'] ?? 'left'));
            return '<' . $level . ' style="margin:0 0 16px;text-align:' . $align . ';">' . $text . '</' . $level . '>';

        case 'button':
            $text = sb_public_h((string)($content['text'] ?? 'Кнопка'));
            $href = sb_public_h((string)($content['href'] ?? '#'));
            $target = sb_public_h((string)($content['target'] ?? '_self'));
            $align = sb_public_h((string)($content['align'] ?? 'left'));
            return '<div style="margin:0 0 16px;text-align:' . $align . ';"><a href="' . $href . '" target="' . $target . '" style="display:inline-block;padding:10px 16px;border-radius:10px;background:#2563eb;color:#fff;text-decoration:none;font-weight:600;">' . $text . '</a></div>';

        case 'html':
            return '<div style="margin:0 0 16px;">' . (string)($content['html'] ?? '') . '</div>';

        case 'spacer':
            $height = max(0, (int)($content['height'] ?? 30));
            return '<div style="height:' . $height . 'px;"></div>';

        case 'text':
        default:
            return '<div style="margin:0 0 16px;line-height:1.6;">' . (string)($content['html'] ?? '') . '</div>';
    }
}

function sb_public_render_zone(array $blocks): string
{
    if (!$blocks) {
        return '';
    }

    $html = '';
    foreach ($blocks as $block) {
        $html .= sb_public_render_block($block);
    }
    return $html;
}

$site = $siteId > 0 ? sb_public_find_site($siteId) : null;
if (!$site) {
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

$pages = sb_public_pages_for_site((int)$site['id']);
$currentPage = null;

$pageId = (int)($_GET['pageId'] ?? 0);
if ($pageId > 0) {
    $currentPage = sb_public_find_page_for_site((int)$site['id'], $pageId);
}

if (!$currentPage) {
    $homePageId = (int)($site['homePageId'] ?? 0);
    if ($homePageId > 0) {
        $currentPage = sb_public_find_page_for_site((int)$site['id'], $homePageId);
    }
}

if (!$currentPage && !empty($pages)) {
    $currentPage = $pages[0];
}

$layout = sb_public_layout_for_site((int)$site['id']);
$pageBlocks = $currentPage ? sb_public_page_blocks((int)$currentPage['id']) : [];

$settings = isset($site['settings']) && is_array($site['settings']) ? $site['settings'] : [];
$containerWidth = max(320, min(1920, (int)($settings['containerWidth'] ?? 1100)));
$accent = (string)($settings['accent'] ?? '#2563eb');
if (!preg_match('/^#[0-9a-fA-F]{6}$/', $accent)) {
    $accent = '#2563eb';
}

$layoutSettings = isset($layout['settings']) && is_array($layout['settings']) ? $layout['settings'] : [];
$showHeader = !empty($layoutSettings['showHeader']);
$showFooter = !empty($layoutSettings['showFooter']);
$showLeft = !empty($layoutSettings['showLeft']);
$showRight = !empty($layoutSettings['showRight']);
$leftWidth = max(120, min(800, (int)($layoutSettings['leftWidth'] ?? 260)));
$rightWidth = max(120, min(800, (int)($layoutSettings['rightWidth'] ?? 260)));

$headerHtml = sb_public_render_zone($layout['zones']['header'] ?? []);
$footerHtml = sb_public_render_zone($layout['zones']['footer'] ?? []);
$leftHtml = sb_public_render_zone($layout['zones']['left'] ?? []);
$rightHtml = sb_public_render_zone($layout['zones']['right'] ?? []);
$pageHtml = sb_public_render_zone($pageBlocks);
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title><?= sb_public_h((string)($currentPage['title'] ?? $site['name'] ?? 'SiteBuilder')) ?></title>
    <style>
        * { box-sizing: border-box; }
        body {
            margin: 0;
            font-family: Arial, sans-serif;
            color: #1f2937;
            background: #ffffff;
        }
        .site-shell {
            min-height: 100vh;
            display: flex;
            flex-direction: column;
        }
        .site-header,
        .site-footer {
            border-bottom: 1px solid #e5e7eb;
            background: #fff;
        }
        .site-footer {
            border-top: 1px solid #e5e7eb;
            border-bottom: 0;
            margin-top: auto;
        }
        .container {
            width: 100%;
            max-width: <?= (int)$containerWidth ?>px;
            margin: 0 auto;
            padding: 20px;
        }
        .site-nav {
            margin: 0 0 16px;
            display: flex;
            flex-wrap: wrap;
            gap: 10px;
        }
        .site-nav a {
            text-decoration: none;
            color: <?= sb_public_h($accent) ?>;
            font-weight: 600;
        }
        .content-layout {
            display: grid;
            grid-template-columns:
                <?= $showLeft ? (int)$leftWidth . 'px ' : '' ?>
                1fr
                <?= $showRight ? ' ' . (int)$rightWidth . 'px' : '' ?>;
            gap: 20px;
            align-items: start;
        }
        .sidebar,
        .main-content {
            min-width: 0;
        }
        .sidebar-box,
        .content-box {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 16px;
        }
        .page-title {
            margin: 0 0 20px;
            font-size: 32px;
            line-height: 1.2;
        }
        .empty {
            padding: 20px;
            border: 1px dashed #d1d5db;
            border-radius: 12px;
            color: #6b7280;
        }
        @media (max-width: 1000px) {
            .content-layout {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
<div class="site-shell">
    <?php if ($showHeader): ?>
        <header class="site-header">
            <div class="container">
                <?php if ($headerHtml !== ''): ?>
                    <?= $headerHtml ?>
                <?php else: ?>
                    <div style="font-size:24px;font-weight:700;color:<?= sb_public_h($accent) ?>;">
                        <?= sb_public_h((string)($site['name'] ?? 'SiteBuilder')) ?>
                    </div>
                <?php endif; ?>
            </div>
        </header>
    <?php endif; ?>

    <main class="container" style="flex:1 1 auto;">
        <?php if (!empty($pages)): ?>
            <nav class="site-nav">
                <?php foreach ($pages as $page): ?>
                    <a href="<?= sb_public_h($basePath) ?>/public.php?siteId=<?= (int)$site['id'] ?>&pageId=<?= (int)$page['id'] ?>">
                        <?= sb_public_h((string)($page['title'] ?? 'Страница')) ?>
                    </a>
                <?php endforeach; ?>
            </nav>
        <?php endif; ?>

        <div class="content-layout">
            <?php if ($showLeft): ?>
                <aside class="sidebar">
                    <div class="sidebar-box">
                        <?= $leftHtml !== '' ? $leftHtml : '<div class="empty">Левая зона пуста</div>' ?>
                    </div>
                </aside>
            <?php endif; ?>

            <section class="main-content">
                <div class="content-box">
                    <?php if ($currentPage): ?>
                        <h1 class="page-title"><?= sb_public_h((string)($currentPage['title'] ?? 'Страница')) ?></h1>
                        <?= $pageHtml !== '' ? $pageHtml : '<div class="empty">На странице пока нет блоков</div>' ?>
                    <?php else: ?>
                        <div class="empty">У сайта пока нет страниц</div>
                    <?php endif; ?>
                </div>
            </section>

            <?php if ($showRight): ?>
                <aside class="sidebar">
                    <div class="sidebar-box">
                        <?= $rightHtml !== '' ? $rightHtml : '<div class="empty">Правая зона пуста</div>' ?>
                    </div>
                </aside>
            <?php endif; ?>
        </div>
    </main>

    <?php if ($showFooter): ?>
        <footer class="site-footer">
            <div class="container">
                <?= $footerHtml !== '' ? $footerHtml : '<div style="color:#6b7280;">© ' . date('Y') . ' ' . sb_public_h((string)($site['name'] ?? 'SiteBuilder')) . '</div>' ?>
            </div>
        </footer>
    <?php endif; ?>
</div>
</body>
</html>


---

4. Что проверить

Открой:

/local/sitebuilder/public.php?siteId=1

Проверь:

1. открывается ли сайт


2. берётся ли первая/домашняя страница


3. рендерятся ли page blocks


4. рендерятся ли header/footer/left/right


5. работает ли переключение между страницами через меню сверху




---

Что это даст

После этого у тебя уже будет полный базовый цикл:

в editor.php редактируешь страницы и блоки

в layout.php редактируешь зоны layout

в public.php смотришь результат



---

Следующий правильный этап

На этом этапе каркас проекта уже собран.
Дальше уже лучше не плодить страницы, а переходить к нормализации структуры фронта:

1. вынести общий CSS


2. вынести общий JS API client


3. разрезать большие inline-скрипты по файлам


4. сделать нормальную папочную структуру для frontend/admin/public


5. потом уже “сделать красиво”



Следующим сообщением я могу дать тебе нормальную файловую структуру для уже собранного проекта и первый шаг по выносу JS/CSS из страниц в отдельные файлы.