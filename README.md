Да, ты прав. Я сломал визуал тем, что заменил public_page.php целиком не от твоей рабочей версии. Надо делать не так.

Сейчас делаем нормально: верни public_page.php из архива, тот рабочий, который ты прислал, и в него внеси только точечные изменения.

1. В public_page.php найди это место

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди строку:

$appearance = sb_public_appearance_get($site, $vm);

Прямо перед ней вставь этот код:

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
                foreach (['getByPage', 'listByPage', 'getList', 'getByPageId', 'getForPage'] as $method) {
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
            return [];
        }

        $sections = array_values(array_filter($sections, static function ($section) use ($siteId, $pageId) {
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

        foreach ($sections as &$section) {
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

            $section['id'] = (int)($section['id'] ?? 0);
            $section['siteId'] = (int)($section['siteId'] ?? $section['site_id'] ?? $siteId);
            $section['pageId'] = (int)($section['pageId'] ?? $section['page_id'] ?? $pageId);
            $section['title'] = (string)($section['title'] ?? 'Секция');
            $section['sort'] = (int)($section['sort'] ?? 500);
            $section['layout'] = $layout;
            $section['props'] = $props;
        }
        unset($section);

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

            $layout = sb_public_to_array($section['layout'] ?? []);
            $columns = sb_public_clamp_int($layout['columns'] ?? 1, 1, 4);
            $gap = sb_public_clamp_int($layout['gap'] ?? 24, 0, 120);

            $sectionBlocks = $blocksBySection[$sectionId] ?? [];
            $columnBlocks = sb_public_group_blocks_by_column($sectionBlocks, $columns);

            $html .= '<section class="sb-page-section sb-page-section--columns-' . $columns . '">';
            $html .= '<div class="sb-page-section__grid" style="--sb-section-columns:' . $columns . ';--sb-section-gap:' . $gap . 'px;">';

            for ($column = 1; $column <= $columns; $column++) {
                $html .= '<div class="sb-page-section__column sb-page-section__column--' . $column . '">';
                $html .= sb_public_render_blocks($columnBlocks[$column] ?? [], $vm);
                $html .= '</div>';
            }

            $html .= '</div>';
            $html .= '</section>';
        }

        $html .= '</div>';

        return $html;
    }
}

2. В этом же public_page.php найди строку

$pageHtml = sb_public_render_blocks($pageBlocks, $vm);

Замени только её на:

$currentPageId = (int)($currentPage['id'] ?? 0);

$pageSections = $currentPageId > 0
    ? sb_public_page_sections_for_page($vm, $siteId, $currentPageId)
    : [];

$pageHtml = sb_public_render_page_sections($pageSections, $pageBlocks, $vm);

Больше в public_page.php визуал не трогаем.

3. В public.css добавь только это в конец

Файл:

/local/sitebuilder/assets/public/public.css

/* Page sections: columns */
.sb-page-sections {
    width: 100%;
    min-width: 0;
}

.sb-page-section {
    width: 100%;
    min-width: 0;
    box-sizing: border-box;
}

.sb-page-section + .sb-page-section {
    margin-top: 24px;
}

.sb-page-section__grid {
    display: grid;
    grid-template-columns: repeat(var(--sb-section-columns, 1), minmax(0, 1fr));
    gap: var(--sb-section-gap, 24px);
    width: 100%;
    min-width: 0;
    align-items: start;
    box-sizing: border-box;
}

.sb-page-section__column {
    min-width: 0;
    box-sizing: border-box;
}

@media (max-width: 900px) {
    .sb-page-section__grid {
        grid-template-columns: 1fr;
    }
}

4. В public_render.php оставь только фикс для текста

Файл:

/local/sitebuilder/lib/public_render.php

Если ещё не сделал, после sb_public_h() добавь:

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

И в sb_public_render_block() замени:

$content = (array)($block['content'] ?? []);
$props = (array)($block['props'] ?? []);

на:

$content = sb_public_to_array($block['content'] ?? []);
$props = sb_public_to_array($block['props'] ?? []);

И всё. Не заменяем весь шаблон. Так мы возвращаем твой визуал и добавляем только нужное: секции, колонки и нормальный вывод текста.