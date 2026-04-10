Отлично. Раз основные админские страницы уже привели в порядок, следующий правильный шаг — подтянуть публичную часть, чтобы сайт выглядел аккуратнее и более “прикладным”, а не как сырая демо-страница.

Логично сделать это так:

1. обновить public.css


2. слегка улучшить шаблон views/layout/public_page.php



Начнём с самого безопасного и заметного — полного public.css.

Полный файл /local/sitebuilder/assets/public/public.css

* {
    box-sizing: border-box;
}

html,
body {
    margin: 0;
    padding: 0;
}

body {
    font-family: Arial, sans-serif;
    color: #1f2937;
    background: #f5f7fb;
    line-height: 1.5;
}

a {
    color: var(--sb-accent);
}

.sb-public-shell {
    min-height: 100vh;
    display: flex;
    flex-direction: column;
}

.sb-container {
    width: 100%;
    max-width: var(--sb-container-width);
    margin: 0 auto;
    padding: 0 20px;
}

.sb-public-header,
.sb-public-footer {
    background: #ffffff;
    border-bottom: 1px solid #e5e7eb;
}

.sb-public-footer {
    border-top: 1px solid #e5e7eb;
    border-bottom: 0;
    margin-top: auto;
}

.sb-public-header .sb-container,
.sb-public-footer .sb-container {
    padding-top: 18px;
    padding-bottom: 18px;
}

.sb-public-main .sb-container {
    padding-top: 24px;
    padding-bottom: 24px;
}

.sb-brand {
    font-size: 28px;
    font-weight: 700;
    color: var(--sb-accent);
    margin-bottom: 14px;
}

.sb-public-menu {
    display: flex;
    flex-wrap: wrap;
    gap: 10px;
}

.sb-public-menu__link {
    display: inline-flex;
    align-items: center;
    min-height: 38px;
    text-decoration: none;
    color: var(--sb-accent);
    font-weight: 600;
    padding: 8px 12px;
    border-radius: 10px;
    transition: background .15s ease, color .15s ease;
}

.sb-public-menu__link:hover {
    background: #eef2ff;
}

.sb-layout {
    display: grid;
    grid-template-columns: 1fr;
    gap: 20px;
    align-items: start;
}

.sb-layout.sb-layout--left {
    grid-template-columns: var(--sb-left-width) 1fr;
}

.sb-layout.sb-layout--right {
    grid-template-columns: 1fr var(--sb-right-width);
}

.sb-layout.sb-layout--left.sb-layout--right {
    grid-template-columns: var(--sb-left-width) 1fr var(--sb-right-width);
}

.sb-sidebar,
.sb-content {
    min-width: 0;
}

.sb-box {
    background: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    padding: 18px;
    box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
}

.sb-box--content {
    padding: 24px;
}

.sb-page-title {
    margin: 0 0 24px;
    font-size: 34px;
    line-height: 1.15;
    font-weight: 700;
    color: #111827;
}

.sb-block {
    margin: 0 0 18px;
}

.sb-block:last-child {
    margin-bottom: 0;
}

.sb-block__inner {
    min-width: 0;
}

.sb-text {
    font-size: 16px;
    line-height: 1.7;
    color: #374151;
}

.sb-text p {
    margin: 0 0 14px;
}

.sb-text p:last-child {
    margin-bottom: 0;
}

.sb-text ul,
.sb-text ol {
    margin: 0 0 14px 20px;
    padding: 0;
}

.sb-text li + li {
    margin-top: 6px;
}

.sb-heading {
    margin: 0 0 12px;
    line-height: 1.2;
    color: #111827;
}

.sb-heading--h1 {
    font-size: 40px;
}

.sb-heading--h2 {
    font-size: 32px;
}

.sb-heading--h3 {
    font-size: 26px;
}

.sb-heading--h4 {
    font-size: 22px;
}

.sb-heading--h5 {
    font-size: 18px;
}

.sb-heading--h6 {
    font-size: 16px;
}

.sb-button-wrap {
    margin: 0;
}

.sb-button {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-height: 42px;
    padding: 12px 18px;
    border-radius: 12px;
    background: var(--sb-accent);
    color: #ffffff;
    text-decoration: none;
    font-weight: 700;
    transition: opacity .15s ease, transform .15s ease;
}

.sb-button:hover {
    opacity: 0.94;
    transform: translateY(-1px);
}

.sb-empty {
    padding: 18px;
    border: 1px dashed #d1d5db;
    border-radius: 12px;
    color: #6b7280;
    background: #ffffff;
}

.sb-footer-note {
    color: #6b7280;
    font-size: 14px;
}

.sb-sidebar .sb-public-menu {
    display: flex;
    flex-direction: column;
    gap: 6px;
}

.sb-sidebar .sb-public-menu__link {
    width: 100%;
    border-radius: 10px;
    padding: 10px 12px;
}

.sb-sidebar .sb-public-menu__link:hover {
    background: #f3f4f6;
}

.sb-block--html table {
    width: 100%;
    border-collapse: collapse;
    margin: 0 0 14px;
    font-size: 14px;
}

.sb-block--html table th,
.sb-block--html table td {
    border: 1px solid #d1d5db;
    padding: 10px 12px;
    text-align: left;
    vertical-align: top;
}

.sb-block--html table th {
    background: #f9fafb;
    font-weight: 700;
}

.sb-block--html img {
    max-width: 100%;
    height: auto;
    border-radius: 12px;
}

.sb-block--html iframe {
    max-width: 100%;
}

.sb-block--html pre {
    overflow: auto;
    background: #111827;
    color: #f9fafb;
    padding: 14px;
    border-radius: 12px;
    font-size: 13px;
}

.sb-block--html code {
    font-family: Consolas, Monaco, monospace;
}

@media (max-width: 1000px) {
    .sb-layout,
    .sb-layout.sb-layout--left,
    .sb-layout.sb-layout--right,
    .sb-layout.sb-layout--left.sb-layout--right {
        grid-template-columns: 1fr;
    }

    .sb-page-title {
        font-size: 28px;
    }

    .sb-heading--h1 {
        font-size: 32px;
    }

    .sb-heading--h2 {
        font-size: 26px;
    }

    .sb-box,
    .sb-box--content {
        padding: 18px;
    }
}

@media (max-width: 640px) {
    .sb-container {
        padding: 0 14px;
    }

    .sb-public-header .sb-container,
    .sb-public-footer .sb-container {
        padding-top: 14px;
        padding-bottom: 14px;
    }

    .sb-public-main .sb-container {
        padding-top: 16px;
        padding-bottom: 16px;
    }

    .sb-brand {
        font-size: 24px;
    }

    .sb-page-title {
        font-size: 24px;
        margin-bottom: 18px;
    }

    .sb-public-menu {
        gap: 8px;
    }

    .sb-public-menu__link {
        min-height: 34px;
        padding: 7px 10px;
        font-size: 14px;
    }
}

Что это даст

После замены public.css:

шапка и футер станут аккуратнее

меню будет выглядеть лучше

текст и HTML-блоки будут читабельнее

таблицы в html станут нормальными

страница в целом будет выглядеть как рабочий внутренний сайт, а не просто рендер данных


Что проверить

Открой снова:

/local/sitebuilder/public.php?siteId=1

Проверь:

1. улучшились ли отступы


2. лучше ли читается текст


3. аккуратнее ли выглядит меню


4. не сломался ли вывод текущих блоков



Следующим пакетом логично прислать уже улучшенный views/layout/public_page.php, чтобы верхняя часть сайта и контентный контейнер выглядели ещё аккуратнее.