Да, строка могла отличаться по пробелам/переносам. Тогда проще: замени целиком функцию sb_public_render_page_sections() в файле:

/local/sitebuilder/views/layout/public_page.php

Найди:

if (!function_exists('sb_public_render_page_sections')) {

и замени весь этот блок до закрывающей } функции на этот готовый вариант:

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

            $gridStyle = implode('', [
                '--sb-section-columns:' . $columns . ';',
                '--sb-section-gap:' . $gap . 'px;',
                'display:grid !important;',
                'grid-template-columns:repeat(' . $columns . ',minmax(0,1fr)) !important;',
                'gap:' . $gap . 'px !important;',
                'width:100%;',
                'min-width:0;',
                'align-items:start;',
                'box-sizing:border-box;',
            ]);

            $html .= '<section class="sb-page-section sb-page-section--columns-' . $columns . '">';
            $html .= '<div class="sb-page-section__grid" style="' . sb_public_h($gridStyle) . '">';

            for ($column = 1; $column <= $columns; $column++) {
                $html .= '<div class="sb-page-section__column sb-page-section__column--' . $column . '" style="min-width:0;box-sizing:border-box;">';
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

Теперь обязательно в public.css найди и удали/закомментируй этот кусок, если он есть:

@media (max-width: 900px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

И замени на:

@media (max-width: 640px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

После этого в Ctrl+U у секции должно быть:

grid-template-columns:repeat(3,minmax(0,1fr)) !important;

И тогда CSS уже не сможет принудительно сложить всё в одну колонку на обычном экране.