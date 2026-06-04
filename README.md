Да, это уже ошибка публичного CSS/рендера секций. Сейчас публичная страница, скорее всего, задаёт колонки через классы типа:

.sb-page-section--columns-1 .sb-section-grid { ... }
.sb-page-section--columns-2 .sb-section-grid { ... }

И где-то первая секция влияет на следующие. Самый надёжный вариант — задавать количество колонок прямо каждой сетке через CSS-переменную.

1. В public_page.php найди рендер секций

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди функцию:

sb_public_render_page_sections

Внутри неё должен быть кусок, где считается:

$columns = ...

и потом выводится:

<div class="sb-section-grid">

Нужно сделать так, чтобы у каждой секции сетка получала своё число колонок.

Пример правильного куска внутри цикла по секциям:

$layout = sb_public_to_array($section['layout'] ?? []);

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

А сам grid должен быть примерно так:

$html .= '<div class="sb-section-grid" style="--sb-section-columns:' . $columns . ';--sb-section-gap:' . $gap . 'px;">';

То есть важно, чтобы было именно:

style="--sb-section-columns:2"

у второй секции, если у неё 2 колонки.


---

2. Полный пример куска рендера секции

Внутри sb_public_render_page_sections() секция должна собираться примерно так:

$layout = sb_public_to_array($section['layout'] ?? []);
$columns = (int)($layout['columns'] ?? 1);
$columns = max(1, min(4, $columns));

$gap = (int)($layout['gap'] ?? 24);
$gap = max(0, min(120, $gap));

$sectionStyle = sb_public_section_style($section);
$containerClass = sb_public_section_container_class($section);

$html .= '<section class="sb-page-section sb-page-section--columns-' . $columns . '" style="' . sb_public_h($sectionStyle) . '">';
$html .= '  <div class="' . sb_public_h($containerClass) . '">';
$html .= '      <div class="sb-section-grid" style="--sb-section-columns:' . $columns . ';--sb-section-gap:' . $gap . 'px;">';

$columnsBlocks = sb_public_group_blocks_by_column($sectionBlocks, $columns);

for ($column = 1; $column <= $columns; $column++) {
    $html .= '<div class="sb-section-column sb-section-column--' . $column . '">';
    $html .= sb_public_render_blocks($columnsBlocks[$column] ?? [], $vm);
    $html .= '</div>';
}

$html .= '      </div>';
$html .= '  </div>';
$html .= '</section>';

Смысл: каждая секция сама говорит своей сетке, сколько у неё колонок.


---

3. Исправь CSS публичной части

Файл:

/local/sitebuilder/assets/public/public.css

В самый конец добавь:

/* =========================================================
   ISOLATED SECTION COLUMNS
   ========================================================= */

.sb-page-section > .sb-section-container > .sb-section-grid {
    display: grid;
    grid-template-columns: repeat(var(--sb-section-columns, 1), minmax(0, 1fr));
    gap: var(--sb-section-gap, 24px);
    width: 100%;
    min-width: 0;
    align-items: start;
    box-sizing: border-box;
}

.sb-page-section > .sb-section-container > .sb-section-grid > .sb-section-column {
    min-width: 0;
    box-sizing: border-box;
}

@media (max-width: 900px) {
    .sb-page-section > .sb-section-container > .sb-section-grid {
        grid-template-columns: 1fr;
    }
}

Это перекроет старую логику и не даст первой секции влиять на вторую.


---

4. Лучше удалить старые правила колонок

Если в public.css есть такие правила:

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

можно оставить, но лучше удалить или закомментировать, чтобы не путались. Новый вариант через --sb-section-columns надёжнее.


---

5. Обнови версию CSS

В public_page.php поменяй подключение:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=5">

После этого на публичной странице сделай Ctrl + F5.


---

Что должно получиться

Если первая секция:

1 колонка

а вторая секция:

2 колонки

то публично должно быть:

Первая секция:
[контент на всю ширину]

Вторая секция:
[колонка 1] [колонка 2]

Если после этого всё равно будет одна колонка, пришли мне функцию sb_public_render_page_sections() из public_page.php, и я пришлю тебе её готовую целиком.