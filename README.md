Теперь по твоему Ctrl+U видно: HTML уже правильный.

Вот правильная строка:

<div class="sb-page-section__grid" style="display:grid;grid-template-columns:repeat(3,minmax(0,1fr));gap:24px;...">

Если визуально всё равно одна колонка, значит её ломает CSS, скорее всего вот это правило, которое мы раньше добавляли:

@media (max-width: 900px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

Оно может срабатывать из-за ширины окна/масштаба/открытой панели разработчика.

1. Исправь public_page.php

Найди в /local/sitebuilder/views/layout/public_page.php строку, где собирается grid:

$pageHtml .= '<div class="sb-page-section__grid" style="display:grid;grid-template-columns:repeat(' . $columns . ',minmax(0,1fr));gap:' . $gap . 'px;width:100%;min-width:0;align-items:start;box-sizing:border-box;">';

Замени её на:

$pageHtml .= '<div class="sb-page-section__grid" style="--sb-section-columns:' . $columns . ';--sb-section-gap:' . $gap . 'px;display:grid !important;grid-template-columns:repeat(' . $columns . ',minmax(0,1fr)) !important;gap:' . $gap . 'px !important;width:100%;min-width:0;align-items:start;box-sizing:border-box;">';

Это заставит браузер рисовать именно 3 колонки даже если где-то в CSS стоит !important.

2. Исправь public.css

В конце /local/sitebuilder/assets/public/public.css найди и удали или закомментируй этот блок:

@media (max-width: 900px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

Вместо него добавь более аккуратный вариант:

@media (max-width: 640px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

То есть колонки будут складываться только на телефоне, а не на обычном экране.

3. Обнови версию CSS

В public_page.php найди:

/assets/public/public.css?v=9

Поставь:

/assets/public/public.css?v=10

4. Проверь

После этого сделай Ctrl + F5.

В Ctrl+U должно стать так:

<div class="sb-page-section__grid" style="--sb-section-columns:3;--sb-section-gap:24px;display:grid !important;grid-template-columns:repeat(3,minmax(0,1fr)) !important;gap:24px !important;...">

Если после этого всё равно будет одна колонка, значит у тебя окно браузера меньше 640px или включён очень большой масштаб страницы. Но по коду сетка уже точно будет принудительно в 3 колонки.