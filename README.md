Да, значит проблема уже не в editor.js, а в public-рендере.

В редакторе layout.columns = 2 читается нормально, а в public_page.php, скорее всего, layout секции приходит как JSON-строка, и код воспринимает его как пустой массив. Поэтому публичная страница думает, что у секции 1 колонка, и выводит всё столбиком.

1. Исправь public_page.php

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди место, где у тебя идут функции секций, и перед функцией:

if (!function_exists('sb_public_section_css_value')) {

добавь:

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


---

2. Замени чтение props в sb_public_section_style

Найди:

$props = is_array($section['props'] ?? null) ? $section['props'] : [];

Замени на:

$props = sb_public_to_array($section['props'] ?? []);


---

3. Замени чтение layout в sb_public_section_container_class

Найди:

$layout = is_array($section['layout'] ?? null) ? $section['layout'] : [];

Замени на:

$layout = sb_public_to_array($section['layout'] ?? []);


---

4. Замени чтение layout в sb_public_render_page_sections

Найди внутри функции sb_public_render_page_sections:

$layout = is_array($section['layout'] ?? null) ? $section['layout'] : [];
$columns = (int)($layout['columns'] ?? 1);

Замени на:

$layout = sb_public_to_array($section['layout'] ?? []);
$columns = (int)($layout['columns'] ?? 1);


---

5. Проверь группировку блоков по колонкам

В функции:

function sb_public_group_blocks_by_column(array $blocks, int $columns): array

должно быть так:

if (!function_exists('sb_public_group_blocks_by_column')) {
    function sb_public_group_blocks_by_column(array $blocks, int $columns): array
    {
        $columns = max(1, min(4, $columns));

        $result = [];

        for ($i = 1; $i <= $columns; $i++) {
            $result[$i] = [];
        }

        foreach ($blocks as $block) {
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

            $column = max(1, min($columns, $column));

            $result[$column][] = $block;
        }

        return $result;
    }
}


---

6. Проверь CSS публичной части

Файл:

/local/sitebuilder/assets/public/public.css

В конец добавь или проверь, что есть:

.sb-page-sections {
    width: 100%;
    min-width: 0;
}

.sb-page-section {
    width: 100%;
    min-width: 0;
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
    align-items: start;
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

@media (max-width: 900px) {
    .sb-page-section--columns-2 .sb-section-grid,
    .sb-page-section--columns-3 .sb-section-grid,
    .sb-page-section--columns-4 .sb-section-grid {
        grid-template-columns: 1fr;
    }
}


---

7. Обнови версию CSS в public_page.php

Найди:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css">

или текущую версию, например:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=2">

Поставь новую:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=3">


---

После этого сделай Ctrl + F5 на публичной странице.

Должно стать так:

Основная секция
├── колонка 1: button
└── колонка 2: button + heading

Если всё равно будет столбиком, пришли мне кусок из public_page.php, где у тебя функции:

sb_public_render_page_sections
sb_public_group_blocks_by_section
sb_public_group_blocks_by_column

Там уже точечно поправим.