Смотрел архив. У тебя нужная функция есть, но не в public_page.php, а здесь:

/local/sitebuilder/lib/public_render.php

Именно там надо править. В твоём архиве она примерно на строке 387.

1. Замени функцию в lib/public_render.php

Файл:

/local/sitebuilder/lib/public_render.php

Найди:

if (!function_exists('sb_public_render_page_sections')) {

и замени весь блок функции на этот:

if (!function_exists('sb_public_render_page_sections')) {
    function sb_public_render_page_sections(array $sections, array $pageBlocks, array $context = []): string
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
                'width:100% !important;',
                'min-width:0 !important;',
                'align-items:start !important;',
                'box-sizing:border-box !important;',
            ]);

            $html .= '<section class="sb-page-section sb-page-section--columns-' . $columns . '">';
            $html .= '<div class="sb-page-section__grid" style="' . sb_public_h($gridStyle) . '">';

            for ($column = 1; $column <= $columns; $column++) {
                $columnStyle = implode('', [
                    'min-width:0 !important;',
                    'box-sizing:border-box !important;',
                ]);

                $html .= '<div class="sb-page-section__column sb-page-section__column--' . $column . '" style="' . sb_public_h($columnStyle) . '">';
                $html .= sb_public_render_blocks($columnBlocks[$column] ?? [], $context);
                $html .= '</div>';
            }

            $html .= '</div>';
            $html .= '</section>';
        }

        $html .= '</div>';

        return $html;
    }
}

Главное отличие: теперь grid-template-columns идёт с !important, поэтому CSS больше не сможет сложить 3 колонки в одну.


---

2. В public.css убери складывание колонок на 640px

Файл:

/local/sitebuilder/assets/public/public.css

В самом конце у тебя сейчас есть:

@media (max-width: 640px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

Временно закомментируй его:

/*
@media (max-width: 640px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}
*/

Потом, когда всё заработает, адаптив можно вернуть аккуратнее.


---

3. Обнови версию CSS в public_page.php

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди:

/assets/public/public.css?v=9

замени на:

/assets/public/public.css?v=10


---

4. Проверь синтаксис

На сервере выполни:

php -l /srv/bx/docroot/local/sitebuilder/lib/public_render.php
php -l /srv/bx/docroot/local/sitebuilder/views/layout/public_page.php


---

5. Что должно появиться в Ctrl+U

После Ctrl + F5 в исходнике должно быть:

<div class="sb-page-section__grid" style="--sb-section-columns:3;--sb-section-gap:24px;display:grid !important;grid-template-columns:repeat(3,minmax(0,1fr)) !important;gap:24px !important;...">

Если эта строка есть, секция принудительно рисуется в 3 колонки.