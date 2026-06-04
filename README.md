Идём дальше: подключаем вывод секций на публичной странице.

Теперь сайт будет рендериться так:

Страница
└── Секция 1
    └── Колонка 1
        └── Компоненты / старые блоки

└── Секция 2
    ├── Колонка 1
    └── Колонка 2

Сейчас сделаем безопасно: если секций ещё нет, автоматически создаётся “Основная секция”, и старые блоки попадут в неё.


---

1. В public_page.php добавь подключение репозитория

Файл:

/local/sitebuilder/views/layout/public_page.php

После блока переменных:

$site = $vm['site'];
$pages = $vm['pages'];
$currentPage = $vm['currentPage'];
$pageBlocks = $vm['pageBlocks'];
$layout = $vm['layout'];
$menu = $vm['menu'];
$basePath = $vm['basePath'];
$siteId = (int)$vm['siteId'];

сразу добавь:

$pageSections = [];

$pageSectionRepositoryPath = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageSectionRepository.php';

if (file_exists($pageSectionRepositoryPath)) {
    require_once $pageSectionRepositoryPath;
}

if (
    class_exists('PageSectionRepository') &&
    !empty($currentPage) &&
    (int)($currentPage['id'] ?? 0) > 0
) {
    global $USER;

    $currentUserId = is_object($USER) && method_exists($USER, 'GetID')
        ? (int)$USER->GetID()
        : 0;

    $currentPageId = (int)$currentPage['id'];

    PageSectionRepository::ensureDefaultForPage(
        $siteId,
        $currentPageId,
        $currentUserId
    );

    $pageSections = PageSectionRepository::listForPage($siteId, $currentPageId);
}


---

2. В public_page.php добавь функции рендера секций

Ниже твоих функций меню, например после:

if (!function_exists('sb_public_render_auto_pages_menu')) {
    function sb_public_render_auto_pages_menu(...)

добавь этот блок:

if (!function_exists('sb_public_section_css_value')) {
    function sb_public_section_css_value($value, string $suffix = 'px'): string
    {
        if ($value === null || $value === '') {
            return '';
        }

        if (is_numeric($value)) {
            return (string)((int)$value) . $suffix;
        }

        return (string)$value;
    }
}

if (!function_exists('sb_public_section_style')) {
    function sb_public_section_style(array $section): string
    {
        $props = is_array($section['props'] ?? null) ? $section['props'] : [];

        $styles = [];

        $paddingTop = sb_public_section_css_value($props['paddingTop'] ?? 40);
        if ($paddingTop !== '') {
            $styles[] = 'padding-top:' . $paddingTop;
        }

        $paddingBottom = sb_public_section_css_value($props['paddingBottom'] ?? 40);
        if ($paddingBottom !== '') {
            $styles[] = 'padding-bottom:' . $paddingBottom;
        }

        $minHeight = sb_public_section_css_value($props['minHeight'] ?? 0);
        if ($minHeight !== '' && $minHeight !== '0px') {
            $styles[] = 'min-height:' . $minHeight;
        }

        $backgroundColor = trim((string)($props['backgroundColor'] ?? ''));
        if ($backgroundColor !== '') {
            $styles[] = 'background-color:' . $backgroundColor;
        }

        $backgroundImage = trim((string)($props['backgroundImage'] ?? ''));
        if ($backgroundImage !== '') {
            $styles[] = 'background-image:url(\'' . str_replace("'", "\\'", $backgroundImage) . '\')';
            $styles[] = 'background-size:cover';
            $styles[] = 'background-position:center';
        }

        return implode(';', $styles);
    }
}

if (!function_exists('sb_public_section_container_class')) {
    function sb_public_section_container_class(array $section): string
    {
        $layout = is_array($section['layout'] ?? null) ? $section['layout'] : [];
        $container = (string)($layout['container'] ?? 'default');

        $allowed = ['default', 'wide', 'full'];

        if (!in_array($container, $allowed, true)) {
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
            $sectionId = (int)($block['sectionId'] ?? 0);

            if ($sectionId <= 0 || !isset($result[$sectionId])) {
                $sectionId = $firstSectionId;
            }

            if ($sectionId <= 0) {
                continue;
            }

            $result[$sectionId][] = $block;
        }

        foreach ($result as &$blocks) {
            usort($blocks, static function ($a, $b) {
                $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);

                if ($sortCmp !== 0) {
                    return $sortCmp;
                }

                return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
            });
        }
        unset($blocks);

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
            $column = (int)($block['column'] ?? 1);
            $column = max(1, min($columns, $column));

            $result[$column][] = $block;
        }

        return $result;
    }
}

if (!function_exists('sb_public_render_page_sections')) {
    function sb_public_render_page_sections(array $sections, array $pageBlocks, array $context): string
    {
        if (empty($sections)) {
            return sb_public_render_blocks($pageBlocks, $context);
        }

        $blocksBySection = sb_public_group_blocks_by_section($pageBlocks, $sections);

        $html = '<div class="sb-page-sections">';

        foreach ($sections as $section) {
            $sectionId = (int)($section['id'] ?? 0);

            if ($sectionId <= 0) {
                continue;
            }

            $layout = is_array($section['layout'] ?? null) ? $section['layout'] : [];
            $columns = (int)($layout['columns'] ?? 1);
            $columns = max(1, min(4, $columns));

            $gap = (int)($layout['gap'] ?? 24);
            if ($gap < 0) {
                $gap = 0;
            }

            $sectionBlocks = $blocksBySection[$sectionId] ?? [];
            $blocksByColumn = sb_public_group_blocks_by_column($sectionBlocks, $columns);

            $style = sb_public_section_style($section);
            $containerClass = sb_public_section_container_class($section);

            $html .= '<section class="sb-page-section sb-page-section--columns-' . $columns . '" data-section-id="' . $sectionId . '"' . ($style !== '' ? ' style="' . sb_public_h($style) . '"' : '') . '>';
            $html .= '<div class="' . sb_public_h($containerClass) . '">';
            $html .= '<div class="sb-section-grid" style="--sb-section-gap:' . (int)$gap . 'px">';

            for ($column = 1; $column <= $columns; $column++) {
                $columnBlocks = $blocksByColumn[$column] ?? [];

                $html .= '<div class="sb-section-column sb-section-column--' . $column . '">';

                if (!empty($columnBlocks)) {
                    $html .= sb_public_render_blocks($columnBlocks, $context);
                }

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


---

3. Замени создание $pageHtml

Найди строку:

$pageHtml = sb_public_render_blocks($pageBlocks, $vm);

Замени на:

$pageHtml = sb_public_render_page_sections($pageSections, $pageBlocks, $vm);


---

4. Добавь стили в публичный CSS

Файл:

/local/sitebuilder/assets/public/public.css

В конец добавь:

/* =========================================================
   PAGE SECTIONS / TILDA-LIKE STRUCTURE
   ========================================================= */

.sb-page-sections {
    width: 100%;
    min-width: 0;
}

.sb-page-section {
    width: 100%;
    min-width: 0;
    position: relative;
    background-repeat: no-repeat;
    box-sizing: border-box;
}

.sb-section-container {
    width: 100%;
    min-width: 0;
    margin: 0 auto;
    box-sizing: border-box;
}

.sb-section-container--default {
    max-width: var(--sb-container-width, 1180px);
}

.sb-section-container--wide {
    max-width: 1440px;
}

.sb-section-container--full {
    max-width: none;
}

.sb-section-grid {
    display: grid;
    grid-template-columns: 1fr;
    gap: var(--sb-section-gap, 24px);
    width: 100%;
    min-width: 0;
}

.sb-page-section--columns-1 .sb-section-grid {
    grid-template-columns: 1fr;
}

.sb-page-section--columns-2 .sb-section-grid {
    grid-template-columns: repeat(2, minmax(0, 1fr));
}

.sb-page-section--columns-3 .sb-section-grid {
    grid-template-columns: repeat(3, minmax(0, 1fr));
}

.sb-page-section--columns-4 .sb-section-grid {
    grid-template-columns: repeat(4, minmax(0, 1fr));
}

.sb-section-column {
    min-width: 0;
}

.sb-section-column > *:first-child {
    margin-top: 0;
}

.sb-section-column > *:last-child {
    margin-bottom: 0;
}

@media (max-width: 900px) {
    .sb-page-section--columns-2 .sb-section-grid,
    .sb-page-section--columns-3 .sb-section-grid,
    .sb-page-section--columns-4 .sb-section-grid {
        grid-template-columns: 1fr;
    }
}


---

5. Обнови версию CSS в public_page.php

Найди:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css">

Замени на:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=2">


---

После этого открой публичную страницу. Визуально почти ничего не должно сломаться, но структура уже будет новая:

<div class="sb-page-sections">
  <section class="sb-page-section">
    <div class="sb-section-container">
      <div class="sb-section-grid">
        <div class="sb-section-column">
          <!-- старые блоки -->
        </div>
      </div>
    </div>
  </section>
</div>

Следующий шаг — добавить в editor.php панель секций: добавить секцию, выбрать 1/2/3/4 колонки, переносить компоненты по секциям.