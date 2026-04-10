Отлично. Делаем admin.css пакетом и подключаем его во все текущие админские страницы.

Смысл шага:

убрать дублирование стилей

сделать единый внешний вид

дальше уже спокойно выносить JS по файлам



---

1. Создай файл /local/sitebuilder/assets/admin/admin.css

* {
    box-sizing: border-box;
}

html,
body {
    margin: 0;
    padding: 0;
}

body.sb-admin-body {
    font-family: Arial, sans-serif;
    background: #f6f8fb;
    color: #1f2937;
}

.sb-page {
    max-width: 1480px;
    margin: 0 auto;
    padding: 24px;
}

.sb-topbar {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 16px;
    margin-bottom: 20px;
}

.sb-topbar-left h1,
.sb-title {
    margin: 0;
    font-size: 28px;
    font-weight: 700;
}

.sb-subtitle {
    margin: 6px 0 0;
    color: #6b7280;
    font-size: 14px;
}

.sb-back-link {
    display: inline-block;
    margin-bottom: 8px;
    text-decoration: none;
    color: #1d4ed8;
    font-size: 14px;
}

.sb-userbox {
    font-size: 14px;
    color: #374151;
    background: #fff;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
    padding: 10px 14px;
}

.sb-layout-2 {
    display: grid;
    grid-template-columns: 360px 1fr;
    gap: 20px;
}

.sb-layout-2-wide {
    display: grid;
    grid-template-columns: 420px 1fr;
    gap: 20px;
}

.sb-layout-3 {
    display: grid;
    grid-template-columns: 1fr 420px;
    gap: 20px;
}

.sb-grid-2 {
    display: grid;
    grid-template-columns: repeat(2, 1fr);
    gap: 16px;
}

.sb-grid-4 {
    display: grid;
    grid-template-columns: repeat(4, 1fr);
    gap: 14px;
}

.sb-panel,
.sb-card {
    background: #fff;
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    padding: 18px;
    box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
}

.sb-panel + .sb-panel {
    margin-top: 20px;
}

.sb-panel-title {
    margin: 0 0 14px;
    font-size: 18px;
    font-weight: 700;
}

.sb-toolbar,
.sb-actions,
.sb-form-row {
    display: flex;
    gap: 10px;
    flex-wrap: wrap;
    align-items: center;
}

.sb-form-row.align-end {
    align-items: end;
}

.sb-field {
    display: flex;
    flex-direction: column;
    gap: 6px;
    min-width: 180px;
    flex: 1 1 180px;
}

.sb-field.full {
    grid-column: 1 / -1;
}

.sb-field label {
    font-size: 13px;
    color: #4b5563;
}

.sb-input,
.sb-select,
.sb-textarea {
    width: 100%;
    border: 1px solid #d1d5db;
    border-radius: 10px;
    outline: none;
    background: #fff;
    padding: 10px 12px;
    font: inherit;
}

.sb-input,
.sb-select {
    height: 40px;
}

.sb-textarea {
    min-height: 120px;
    resize: vertical;
}

.sb-input:focus,
.sb-select:focus,
.sb-textarea:focus {
    border-color: #2563eb;
}

.sb-btn {
    height: 40px;
    border: 0;
    border-radius: 10px;
    padding: 0 16px;
    cursor: pointer;
    font-weight: 600;
    text-decoration: none;
    display: inline-flex;
    align-items: center;
    justify-content: center;
}

.sb-btn-small {
    height: 34px;
    padding: 0 12px;
    font-size: 13px;
}

.sb-btn-primary {
    background: #2563eb;
    color: #fff;
}

.sb-btn-primary:hover {
    background: #1d4ed8;
}

.sb-btn-light {
    background: #eef2ff;
    color: #1e3a8a;
}

.sb-btn-light:hover {
    background: #e0e7ff;
}

.sb-btn-danger {
    background: #dc2626;
    color: #fff;
}

.sb-btn-danger:hover {
    background: #b91c1c;
}

.sb-btn-gray {
    background: #f3f4f6;
    color: #374151;
}

.sb-btn-gray:hover {
    background: #e5e7eb;
}

.sb-badge {
    display: inline-flex;
    align-items: center;
    gap: 6px;
    background: #f3f4f6;
    color: #374151;
    border-radius: 999px;
    padding: 4px 10px;
    font-size: 12px;
    white-space: nowrap;
}

.sb-badge-green {
    background: #dcfce7;
    color: #166534;
}

.sb-badge-yellow {
    background: #fef3c7;
    color: #92400e;
}

.sb-meta {
    font-size: 13px;
    color: #6b7280;
    line-height: 1.5;
}

.sb-list {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.sb-item {
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    padding: 12px;
    background: #fafafa;
}

.sb-item.active {
    border-color: #2563eb;
    background: #eff6ff;
}

.sb-item-head {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    align-items: flex-start;
}

.sb-item-title {
    margin: 0 0 6px;
    font-size: 15px;
    font-weight: 700;
}

.sb-preview {
    margin-top: 12px;
    padding: 12px;
    border-radius: 10px;
    background: #fff;
    border: 1px solid #e5e7eb;
}

.sb-empty {
    padding: 20px;
    text-align: center;
    color: #6b7280;
    border: 1px dashed #d1d5db;
    border-radius: 12px;
    background: #fff;
}

.sb-output {
    white-space: pre-wrap;
    background: #0f172a;
    color: #e5e7eb;
    border-radius: 12px;
    padding: 14px;
    min-height: 120px;
    font-family: Consolas, Monaco, monospace;
    font-size: 13px;
    overflow: auto;
}

.sb-hidden {
    display: none !important;
}

@media (max-width: 1200px) {
    .sb-layout-2,
    .sb-layout-2-wide,
    .sb-layout-3,
    .sb-grid-2,
    .sb-grid-4 {
        grid-template-columns: 1fr;
    }
}

@media (max-width: 768px) {
    .sb-topbar {
        flex-direction: column;
        align-items: stretch;
    }

    .sb-page {
        padding: 16px;
    }
}


---

2. Как подключать admin.css в страницах

Во всех админских страницах после ShowHead() добавь:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">

И у <body> поставь класс:

<body class="sb-admin-body">

И основной контейнер страницы замени на:

<div class="sb-page">


---

3. Что заменить в index.php

В <head> после ShowHead()

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">

body

<body class="sb-admin-body">

Корневой контейнер

<div class="sb-page">

Верхний блок

Замени контейнер и классы:

<div class="sb-topbar">
    <div>
        <h1 class="sb-title">SiteBuilder</h1>
        <p class="sb-subtitle">Управление сайтами конструктора</p>
    </div>
    <div class="sb-userbox">
        Пользователь:
        <strong><?= htmlspecialchars((string)$USER->GetLogin(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></strong>
        (ID <?= (int)$USER->GetID() ?>)
    </div>
</div>

Панели

Все блоки вида:

<div class="panel">

замени на:

<div class="sb-panel">

Заголовки панелей

<h2 class="sb-panel-title">

Формы

.form-row → .sb-form-row align-end

.field → .sb-field


Кнопки

.btn → .sb-btn

.btn-primary → .sb-btn-primary

.btn-light → .sb-btn-light

.btn-danger → .sb-btn-danger

.btn-small → .sb-btn-small


Отладка

output замени на класс:

<div id="output" class="sb-output">

Пустые состояния

.empty можешь оставить, но лучше заменить на:

<div class="sb-empty">

Списки/карточки сайтов

.sites-grid можно временно оставить как local style

но карточки лучше перевести на sb-card



---

4. Что заменить в editor.php

В <head>

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">

body

<body class="sb-admin-body">

Корень

<div class="sb-page">

Верхний блок

<div class="sb-topbar">
    <div class="sb-topbar-left">
        <a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← К списку сайтов</a>
        <h1 class="sb-title">Редактор сайта</h1>
        <p class="sb-subtitle">siteId = <?= (int)$siteId ?></p>
    </div>
</div>

Основная сетка

<div class="sb-layout-2">

и внутреннюю правую колонку:

<div class="sb-layout-3">

Панели

.panel → .sb-panel

Заголовки

.panel-title → .sb-panel-title

Списки

.pages-list → .sb-list

.blocks-list → .sb-list


Карточки

.page-card / .block-card → .sb-item

активная страница: sb-item active


Заголовки карточек

.page-title / .block-title → .sb-item-title


Метаданные

.meta → .sb-meta

Preview

.block-preview → .sb-preview

Пустое состояние

.empty → .sb-empty

Отладка

.output → .sb-output

Редактор справа

hidden → sb-hidden

Формы

field → sb-field

input/select/textarea → sb-input/sb-select/sb-textarea


Кнопки

Точно так же заменить на sb-btn*


---

5. Что заменить в menu.php

Подключение такое же:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">

И дальше:

body → sb-admin-body

контейнер → sb-page

topbar → sb-topbar

back-link → sb-back-link

title → sb-title

subtitle → sb-subtitle

panel → sb-panel

panel-title → sb-panel-title

form-row → sb-form-row

field → sb-field

output → sb-output

empty → sb-empty


Карточки меню и пунктов можно пока оставить локальными, потому что там своя вложенная структура.
Это нормально на первом проходе.


---

6. Что заменить в files.php

То же самое:

подключить admin.css

body → sb-admin-body

контейнер → sb-page

topbar/title/subtitle/back-link → sb-*

panel/title → sb-panel, sb-panel-title

toolbar → sb-toolbar

buttons → sb-btn*

meta → sb-meta

empty → sb-empty

output → sb-output


Карточки файлов можно пока оставить локальными.


---

7. Что заменить в settings.php

То же самое:

подключить admin.css

body → sb-admin-body

.page → .sb-page

.topbar → .sb-topbar

.back-link → .sb-back-link

.title → .sb-title

.subtitle → .sb-subtitle

.panel → .sb-panel

.panel-title → .sb-panel-title

.form-grid лучше временно оставить локальным, либо заменить на .sb-grid-2

.field → .sb-field

input/select → .sb-input/.sb-select

.toolbar → .sb-toolbar

.meta → .sb-meta

.output → .sb-output

.empty → .sb-empty



---

8. Что заменить в layout.php

То же самое:

подключить admin.css

body → sb-admin-body

.page → .sb-page

.topbar → .sb-topbar

.back-link → .sb-back-link

.title → .sb-title

.subtitle → .sb-subtitle

.panel → .sb-panel

.panel-title → .sb-panel-title

.settings-grid можно заменить на .sb-grid-4

.field → .sb-field

input/select/textarea → sb-input/sb-select/sb-textarea

.toolbar → .sb-toolbar

.meta → .sb-meta

.output → .sb-output

.hidden → .sb-hidden


Карточки зон и layout-блоков пока можно оставить локальными.


---

9. Практический способ внедрить без ломки

Чтобы не переписывать сразу весь HTML, делай так:

Сначала в каждой странице:

1. подключи admin.css


2. поставь body class="sb-admin-body"


3. замени:

page

topbar

panel

panel-title

output

empty

кнопки

field/input/select/textarea




А локальные уникальные стили:

cards specific

dialog specific

custom grids


оставь пока во встроенном <style>.

Это самый безопасный путь.


---

10. Что я рекомендую прямо сейчас

Чтобы не делать “на глаз”, лучше идти так:

1. сначала привести editor.php на admin.css


2. потом index.php


3. потом menu.php


4. потом остальные



Потому что editor.php — самая важная админская страница, и на ней сразу будет видно эффект.

Напиши: давай приведём сначала editor.php под admin.css