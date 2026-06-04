Да, по скрину видно: секции уже выводятся, но сетка колонок не применяется. Поэтому колонки идут обычными div друг под другом.

Причина почти точно в CSS: в public_page.php сейчас выводится класс:

sb-page-section__grid

а в старом CSS мог остаться класс:

.sb-section-grid

Сделаем универсально, чтобы работали оба варианта классов.

1. В public.css добавь в самый конец

Файл:

/local/sitebuilder/assets/public/public.css

Добавь именно в самый низ:

/* =========================================================
   SITEBUILDER PAGE SECTIONS FIX
   Работает и со старым, и с новым HTML секций
   ========================================================= */

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

/* Новый вариант */
.sb-page-section__grid {
    display: grid !important;
    grid-template-columns: repeat(var(--sb-section-columns, 1), minmax(0, 1fr)) !important;
    gap: var(--sb-section-gap, 24px) !important;
    width: 100%;
    min-width: 0;
    align-items: start;
    box-sizing: border-box;
}

.sb-page-section__column {
    min-width: 0;
    box-sizing: border-box;
}

/* Старый вариант, если где-то остался */
.sb-section-grid {
    display: grid !important;
    grid-template-columns: repeat(var(--sb-section-columns, 1), minmax(0, 1fr)) !important;
    gap: var(--sb-section-gap, 24px) !important;
    width: 100%;
    min-width: 0;
    align-items: start;
    box-sizing: border-box;
}

.sb-section-column {
    min-width: 0;
    box-sizing: border-box;
}

@media (max-width: 900px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

2. Обнови версию CSS в рабочем public_page.php

Найди подключение:

/assets/public/public.css

Например:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=7">

Поставь новую версию:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=8">

Если там без версии, сделай так:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=8">

3. Проверь, что в public_page.php grid выводится так

В твоей рабочей версии, которую мы точечно правили, внутри функции sb_public_render_page_sections() должно быть:

$html .= '<div class="sb-page-section__grid" style="--sb-section-columns:' . $columns . ';--sb-section-gap:' . $gap . 'px;">';

Если у тебя там сейчас другой класс:

$html .= '<div class="sb-section-grid" style="--sb-section-columns:' . $columns . ';--sb-section-gap:' . $gap . 'px;">';

это тоже нормально — CSS выше покрывает оба варианта.

4. Сделай жёсткое обновление

На публичной странице нажми:

Ctrl + F5

Или временно открой с параметром:

/local/sitebuilder/public.php?siteId=13&pageId=13&clear_cache=Y

Что должно получиться

У тебя в редакторе:

Основная секция:
колонка 1 — button
колонка 2 — button
колонка 3 — пусто

Новая секция:
колонка 1 — пусто
колонка 2 — text
колонка 3 — пусто

На публичной странице должно стать так же: первая секция в 3 колонки, ниже вторая секция тоже в 3 колонки, а текст должен стоять во второй колонке, а не идти списком под кнопками.