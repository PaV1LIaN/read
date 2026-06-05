Да, значит у тебя рендер секций сделан не функцией, а прямо внутри public_page.php в месте, где собирается $pageHtml.

Сделай так.

1. В public_page.php найди кусок

Ищи не функцию, а строку:

$pageHtml = '<div class="sb-page-sections">';

или:

sb-page-section__grid

Там будет большой блок примерно такого вида:

$pageHtml = '<div class="sb-page-sections">';

foreach ($pageSections as $section) {
    ...
    $pageHtml .= '<section class="sb-page-section ...">';
    $pageHtml .= '<div class="sb-page-section__grid" style="display:grid;grid-template-columns:repeat(' . $columns . ',minmax(0,1fr));...">';
    ...
}

$pageHtml .= '</div>';

Вот этот весь блок замени на код ниже.

2. Готовый код для замены блока $pageHtml

$pageHtml = '<div class="sb-page-sections">';

$blocksBySection = [];
$firstSectionId = 0;

foreach ($pageSections as $section) {
    $sectionId = (int)($section['id'] ?? 0);

    if ($sectionId <= 0) {
        continue;
    }

    if ($firstSectionId <= 0) {
        $firstSectionId = $sectionId;
    }

    $blocksBySection[$sectionId] = [];
}

foreach ($pageBlocks as $block) {
    $props = $block['props'] ?? [];

    if (is_string($props) && trim($props) !== '') {
        $decodedProps = json_decode($props, true);
        $props = is_array($decodedProps) ? $decodedProps : [];
    }

    if (!is_array($props)) {
        $props = [];
    }

    $placement = $props['_placement'] ?? [];

    if (is_string($placement) && trim($placement) !== '') {
        $decodedPlacement = json_decode($placement, true);
        $placement = is_array($decodedPlacement) ? $decodedPlacement : [];
    }

    if (!is_array($placement)) {
        $placement = [];
    }

    $sectionId = (int)($block['sectionId'] ?? 0);

    if ($sectionId <= 0) {
        $sectionId = (int)($props['sectionId'] ?? 0);
    }

    if ($sectionId <= 0) {
        $sectionId = (int)($placement['sectionId'] ?? 0);
    }

    if ($sectionId <= 0 || !isset($blocksBySection[$sectionId])) {
        $sectionId = $firstSectionId;
    }

    if ($sectionId > 0 && isset($blocksBySection[$sectionId])) {
        $blocksBySection[$sectionId][] = $block;
    }
}

foreach ($pageSections as $section) {
    $sectionId = (int)($section['id'] ?? 0);

    if ($sectionId <= 0) {
        continue;
    }

    $layout = $section['layout'] ?? [];

    if (is_string($layout) && trim($layout) !== '') {
        $decodedLayout = json_decode($layout, true);
        $layout = is_array($decodedLayout) ? $decodedLayout : [];
    }

    if (!is_array($layout)) {
        $layout = [];
    }

    $columns = (int)($layout['columns'] ?? 1);

    if ($columns < 1) {
        $columns = 1;
    }

    if ($columns > 4) {
        $columns = 4;
    }

    $gap = (int)($layout['gap'] ?? 24);

    if ($gap < 0) {
        $gap = 0;
    }

    if ($gap > 120) {
        $gap = 120;
    }

    $sectionBlocks = $blocksBySection[$sectionId] ?? [];

    $columnBlocks = [];

    for ($i = 1; $i <= $columns; $i++) {
        $columnBlocks[$i] = [];
    }

    foreach ($sectionBlocks as $block) {
        $props = $block['props'] ?? [];

        if (is_string($props) && trim($props) !== '') {
            $decodedProps = json_decode($props, true);
            $props = is_array($decodedProps) ? $decodedProps : [];
        }

        if (!is_array($props)) {
            $props = [];
        }

        $placement = $props['_placement'] ?? [];

        if (is_string($placement) && trim($placement) !== '') {
            $decodedPlacement = json_decode($placement, true);
            $placement = is_array($decodedPlacement) ? $decodedPlacement : [];
        }

        if (!is_array($placement)) {
            $placement = [];
        }

        $column = (int)($block['column'] ?? 0);

        if ($column <= 0) {
            $column = (int)($props['column'] ?? 0);
        }

        if ($column <= 0) {
            $column = (int)($placement['column'] ?? 0);
        }

        if ($column < 1) {
            $column = 1;
        }

        if ($column > $columns) {
            $column = $columns;
        }

        $columnBlocks[$column][] = $block;
    }

    $gridStyle = implode('', [
        'display:grid !important;',
        'grid-template-columns:repeat(' . $columns . ',minmax(0,1fr)) !important;',
        'gap:' . $gap . 'px !important;',
        'width:100%;',
        'min-width:0;',
        'align-items:start;',
        'box-sizing:border-box;',
    ]);

    $pageHtml .= '<section class="sb-page-section sb-page-section--columns-' . $columns . '">';
    $pageHtml .= '<div class="sb-page-section__grid" style="' . htmlspecialchars($gridStyle, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') . '">';

    for ($column = 1; $column <= $columns; $column++) {
        $pageHtml .= '<div class="sb-page-section__column sb-page-section__column--' . $column . '" style="min-width:0;box-sizing:border-box;">';
        $pageHtml .= sb_public_render_blocks($columnBlocks[$column] ?? [], $vm);
        $pageHtml .= '</div>';
    }

    $pageHtml .= '</div>';
    $pageHtml .= '</section>';
}

$pageHtml .= '</div>';

3. В public.css обязательно убери старый media-блок

Найди в /local/sitebuilder/assets/public/public.css:

@media (max-width: 900px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

Замени на:

@media (max-width: 640px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

4. После замены проверь

Выполни:

php -l /srv/bx/docroot/local/sitebuilder/views/layout/public_page.php

Потом Ctrl + F5.

В Ctrl+U теперь должно быть:

grid-template-columns:repeat(3,minmax(0,1fr)) !important;

Если это появится, а визуально всё равно одна колонка — значит экран/контейнер реально очень узкий или другой CSS перебивает ширину колонок, но текущий рендер уже будет принудительно отдавать 3 колонки.