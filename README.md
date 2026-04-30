Ошибка из-за конфликта функций.

У тебя уже подключён файл:

/local/sitebuilder/lib/public_render.php

и в нём уже есть функция:

sb_public_render_blocks(array $blocks, array $context)

А в присланном public_page.php вызов получился старого формата:

sb_public_render_blocks($blocks, $siteId, $currentPageId, $basePath)

Поэтому PHP и ругается: вторым аргументом ждёт array, а получил int.

Быстрое исправление

В файле:

/local/sitebuilder/views/layout/public_page.php

найди место перед рендером страницы, где у тебя уже есть:

$currentPageId = (int)($currentPage['id'] ?? 0);

Сразу ниже добавь:

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

Теперь найди строку примерно на 564:

<?= sb_public_render_blocks($blocks, $siteId, $currentPageId, $basePath) ?>

Замени на:

<?= sb_public_render_blocks($blocks, $renderContext) ?>

Дальше найди функцию:

function sb_public_render_layout_blocks(array $layout, string $zone, int $siteId, int $pageId, string $basePath): string

и замени её целиком на эту:

if (!function_exists('sb_public_render_layout_blocks')) {
    function sb_public_render_layout_blocks(array $layout, string $zone, array $context): string
    {
        $zones = is_array($layout['zones'] ?? null) ? $layout['zones'] : [];
        $blocks = is_array($zones[$zone] ?? null) ? $zones[$zone] : [];

        if (empty($blocks)) {
            return '';
        }

        return sb_public_render_blocks($blocks, $context);
    }
}

После этого ниже найди вызовы:

$leftHtml = sb_public_render_layout_blocks($layout, 'left', $siteId, $currentPageId, $basePath);
$rightHtml = sb_public_render_layout_blocks($layout, 'right', $siteId, $currentPageId, $basePath);
$footerHtml = sb_public_render_layout_blocks($layout, 'footer', $siteId, $currentPageId, $basePath);
$headerHtml = sb_public_render_layout_blocks($layout, 'header', $siteId, $currentPageId, $basePath);

Замени на:

$leftHtml = sb_public_render_layout_blocks($layout, 'left', $renderContext);
$rightHtml = sb_public_render_layout_blocks($layout, 'right', $renderContext);
$footerHtml = sb_public_render_layout_blocks($layout, 'footer', $renderContext);
$headerHtml = sb_public_render_layout_blocks($layout, 'header', $renderContext);

Почему так

Мы не должны заново объявлять sb_public_render_blocks(), потому что она уже есть в lib/public_render.php. Нужно просто вызывать её в правильном формате:

sb_public_render_blocks($blocks, $renderContext)

После этих замен ошибка с Argument #2 ($context) must be of type array уйдёт.