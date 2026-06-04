Понял. Если добавление CSS вообще ничего не меняет, значит проблема не в CSS, а в том, что публичная страница всё ещё выводит блоки старым способом:

$pageHtml = sb_public_render_blocks($pageBlocks, $vm);

То есть секции в public вообще не участвуют.

Сделаем жёстко и без поломки визуала: не трогаем весь шаблон, а меняем только рендер $pageHtml.


---

1. В public_page.php найди строку

$pageHtml = sb_public_render_blocks($pageBlocks, $vm);

Замени её на это:

$currentPageId = (int)($currentPage['id'] ?? 0);

$pageSections = [];

$repoFile = dirname(__DIR__, 2) . '/lib/PageSectionRepository.php';

if ($currentPageId > 0 && file_exists($repoFile)) {
    require_once $repoFile;

    if (class_exists('PageSectionRepository')) {
        if (method_exists('PageSectionRepository', 'listByPage')) {
            $pageSections = PageSectionRepository::listByPage($siteId, $currentPageId);
        } elseif (method_exists('PageSectionRepository', 'getByPage')) {
            $pageSections = PageSectionRepository::getByPage($siteId, $currentPageId);
        }
    }
}

if (isset($pageSections['sections']) && is_array($pageSections['sections'])) {
    $pageSections = $pageSections['sections'];
}

if (!is_array($pageSections)) {
    $pageSections = [];
}

$pageSections = array_values(array_filter($pageSections, static function ($section) use ($siteId, $currentPageId) {
    return
        (int)($section['siteId'] ?? $section['site_id'] ?? 0) === $siteId &&
        (int)($section['pageId'] ?? $section['page_id'] ?? 0) === $currentPageId;
}));

usort($pageSections, static function ($a, $b) {
    $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

    if ($sortCmp !== 0) {
        return $sortCmp;
    }

    return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
});

if (!empty($pageSections)) {
    $blocksBySection = [];

    foreach ($pageSections as $section) {
        $sectionId = (int)($section['id'] ?? 0);

        if ($sectionId > 0) {
            $blocksBySection[$sectionId] = [];
        }
    }

    $firstSectionId = (int)($pageSections[0]['id'] ?? 0);

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

    $pageHtml = '<div class="sb-page-sections">';

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

        $pageHtml .= '<section class="sb-page-section">';
        $pageHtml .= '<div class="sb-page-section__grid" style="display:grid;grid-template-columns:repeat(' . $columns . ',minmax(0,1fr));gap:' . $gap . 'px;width:100%;">';

        for ($column = 1; $column <= $columns; $column++) {
            $pageHtml .= '<div class="sb-page-section__column" style="min-width:0;">';
            $pageHtml .= sb_public_render_blocks($columnBlocks[$column] ?? [], $vm);
            $pageHtml .= '</div>';
        }

        $pageHtml .= '</div>';
        $pageHtml .= '</section>';
    }

    $pageHtml .= '</div>';
} else {
    $pageHtml = sb_public_render_blocks($pageBlocks, $vm);
}


---

2. Почему это должно сработать

Тут я специально задал сетку inline-стилем:

style="display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:24px;"

То есть теперь даже если public.css не обновился или где-то конфликтует, колонки всё равно должны появиться.


---

3. Проверь через исходный код

После замены открой публичную страницу, нажми Ctrl + U и найди:

sb-page-section__grid

Должно быть что-то такое:

<div class="sb-page-section__grid" style="display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:24px;width:100%;">

Если такой строки нет — значит ты заменил не тот $pageHtml или страница грузит другой шаблон.

Если строка есть, но визуально всё равно одна колонка — пришли кусок HTML из Ctrl + U вокруг sb-page-section__grid, там будет видно точную причину.