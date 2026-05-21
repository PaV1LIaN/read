Да, делаем вариант 2. Начнём безопасно: сначала улучшим внешний вид диска через CSS, не трогая логику загрузки, удаления, папок и прав.

Заменяй файл:

/local/sitebuilder/components/disk/styles.css

на этот:

/* =========================================================
   SITEBUILDER DISK — MODERN FILE MANAGER
   Вариант 2: красивый корпоративный диск
   ========================================================= */

.sb-disk {
    --disk-accent: var(--sb-accent, #2563eb);
    --disk-bg: #ffffff;
    --disk-soft: #f8fafc;
    --disk-border: #e5e7eb;
    --disk-text: #111827;
    --disk-muted: #6b7280;
    --disk-radius: 18px;
    --disk-shadow: 0 16px 40px rgba(15, 23, 42, .06);

    width: 100%;
    min-width: 0;
    color: var(--disk-text);
}

/* Главная карточка диска */
.sb-disk,
.sb-disk-panel,
.sb-disk-root,
.sb-disk-wrap {
    box-sizing: border-box;
}

/* Если JS рисует внутреннюю оболочку */
.sb-disk > div:first-child:not(.sb-public-disk-loading) {
    width: 100%;
}

/* =========================================================
   HEADER / Верхняя часть
   ========================================================= */

.sb-public-block--disk {
    margin-top: 18px;
}

.sb-public-disk-head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 14px;
}

.sb-public-block-title,
.sb-disk-title,
.sb-disk h2,
.sb-disk h3 {
    margin: 0;
    font-size: 22px;
    line-height: 1.25;
    font-weight: 800;
    color: var(--disk-text);
}

/* Старые мелкие подписи */
.sb-disk-count,
.sb-disk-meta,
.sb-disk-subtitle,
.sb-disk-info {
    margin-top: 4px;
    color: var(--disk-muted);
    font-size: 13px;
    line-height: 1.4;
}

/* =========================================================
   TOOLBAR / Панель действий
   ========================================================= */

.sb-disk-toolbar,
.sb-disk-actions,
.sb-disk-topbar,
.sb-disk-controls,
.sb-disk-panel-actions {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    flex-wrap: wrap;
    margin: 14px 0;
}

.sb-disk-actions-left,
.sb-disk-actions-right,
.sb-disk-toolbar-left,
.sb-disk-toolbar-right {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
}

/* Кнопки внутри диска */
.sb-disk button,
.sb-disk .sb-btn,
.sb-public-block--disk button {
    min-height: 38px;
    padding: 0 14px;
    border: 1px solid var(--disk-border);
    border-radius: 12px;
    background: #fff;
    color: #374151;
    font-size: 13px;
    font-weight: 700;
    cursor: pointer;
    transition:
        background .15s ease,
        border-color .15s ease,
        color .15s ease,
        box-shadow .15s ease,
        transform .15s ease;
}

.sb-disk button:hover,
.sb-disk .sb-btn:hover,
.sb-public-block--disk button:hover {
    border-color: #c7d2fe;
    background: #f8fbff;
    color: var(--disk-accent);
    box-shadow: 0 8px 18px rgba(37, 99, 235, .08);
}

.sb-disk button:active,
.sb-public-block--disk button:active {
    transform: translateY(1px);
}

/* Основная кнопка загрузки */
.sb-disk button[data-action="upload"],
.sb-disk button[data-disk-action="upload"],
.sb-disk .sb-disk-upload-btn,
.sb-disk .sb-btn-primary,
.sb-disk .is-primary {
    border-color: var(--disk-accent);
    background: var(--disk-accent);
    color: #fff;
}

.sb-disk button[data-action="upload"]:hover,
.sb-disk button[data-disk-action="upload"]:hover,
.sb-disk .sb-disk-upload-btn:hover,
.sb-disk .sb-btn-primary:hover,
.sb-disk .is-primary:hover {
    background: var(--disk-accent);
    color: #fff;
    box-shadow: 0 10px 22px rgba(37, 99, 235, .22);
}

/* Кнопки Таблица / Плитка */
.sb-disk-view-toggle,
.sb-disk-view-buttons {
    display: inline-flex;
    align-items: center;
    padding: 3px;
    border: 1px solid var(--disk-border);
    border-radius: 14px;
    background: #f8fafc;
    gap: 3px;
}

.sb-disk-view-toggle button,
.sb-disk-view-buttons button {
    min-height: 32px;
    border: 0;
    border-radius: 10px;
    background: transparent;
    box-shadow: none;
}

.sb-disk-view-toggle button.is-active,
.sb-disk-view-buttons button.is-active,
.sb-disk button.is-active {
    background: #fff;
    color: var(--disk-accent);
    box-shadow: 0 4px 12px rgba(15, 23, 42, .08);
}

/* =========================================================
   SEARCH / FILTERS
   ========================================================= */

.sb-disk-filter,
.sb-disk-filters,
.sb-disk-search-row {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
    margin: 14px 0;
}

.sb-disk input[type="text"],
.sb-disk input[type="search"],
.sb-disk select {
    height: 40px;
    border: 1px solid var(--disk-border);
    border-radius: 12px;
    background: #fff;
    color: #111827;
    font-size: 13px;
    outline: none;
    transition: border-color .15s ease, box-shadow .15s ease;
}

.sb-disk input[type="text"],
.sb-disk input[type="search"] {
    min-width: 260px;
    padding: 0 14px;
}

.sb-disk select {
    min-width: 170px;
    padding: 0 34px 0 12px;
}

.sb-disk input[type="text"]:focus,
.sb-disk input[type="search"]:focus,
.sb-disk select:focus {
    border-color: var(--disk-accent);
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12);
}

/* =========================================================
   BREADCRUMBS / Путь
   ========================================================= */

.sb-disk-breadcrumbs,
.sb-disk-path {
    display: flex;
    align-items: center;
    gap: 6px;
    flex-wrap: wrap;
    margin: 10px 0 14px;
    color: var(--disk-muted);
    font-size: 13px;
}

.sb-disk-breadcrumbs a,
.sb-disk-path a {
    display: inline-flex;
    align-items: center;
    min-height: 28px;
    padding: 0 9px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #374151;
    text-decoration: none;
    font-weight: 700;
}

.sb-disk-breadcrumbs a:hover,
.sb-disk-path a:hover {
    background: #eef2ff;
    color: var(--disk-accent);
}

/* =========================================================
   TABLE VIEW
   ========================================================= */

.sb-disk-table-wrap,
.sb-disk-table-container {
    width: 100%;
    overflow-x: auto;
    border: 1px solid var(--disk-border);
    border-radius: 16px;
    background: #fff;
}

.sb-disk table {
    width: 100%;
    border-collapse: separate;
    border-spacing: 0;
    background: #fff;
    min-width: 720px;
}

.sb-disk thead th {
    height: 44px;
    padding: 0 14px;
    border-bottom: 1px solid var(--disk-border);
    background: #f8fafc;
    color: #64748b;
    font-size: 12px;
    font-weight: 800;
    text-align: left;
    text-transform: uppercase;
    letter-spacing: .03em;
}

.sb-disk tbody td {
    height: 54px;
    padding: 10px 14px;
    border-bottom: 1px solid #f1f5f9;
    color: #374151;
    font-size: 13px;
    vertical-align: middle;
}

.sb-disk tbody tr:last-child td {
    border-bottom: 0;
}

.sb-disk tbody tr {
    transition: background .15s ease;
}

.sb-disk tbody tr:hover {
    background: #f8fbff;
}

/* Название файла */
.sb-disk-file-name,
.sb-disk-name,
.sb-disk-item-name,
.sb-disk td:first-child {
    font-weight: 700;
    color: #111827;
}

/* =========================================================
   FILE/FOLDER ICONS
   ========================================================= */

.sb-disk-icon,
.sb-disk-file-icon,
.sb-disk-folder-icon {
    width: 34px;
    height: 34px;
    border-radius: 12px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    margin-right: 10px;
    background: #eef2ff;
    color: var(--disk-accent);
    font-weight: 900;
    flex: 0 0 auto;
}

.sb-disk-folder-icon,
.sb-disk-icon-folder,
[data-type="folder"] .sb-disk-icon,
.sb-disk-row-folder .sb-disk-icon {
    background: #fef3c7;
    color: #92400e;
}

.sb-disk-icon-pdf,
[data-ext="pdf"] .sb-disk-icon {
    background: #fee2e2;
    color: #991b1b;
}

.sb-disk-icon-doc,
.sb-disk-icon-docx,
[data-ext="doc"] .sb-disk-icon,
[data-ext="docx"] .sb-disk-icon {
    background: #dbeafe;
    color: #1d4ed8;
}

.sb-disk-icon-xls,
.sb-disk-icon-xlsx,
[data-ext="xls"] .sb-disk-icon,
[data-ext="xlsx"] .sb-disk-icon {
    background: #dcfce7;
    color: #166534;
}

.sb-disk-icon-img,
[data-ext="jpg"] .sb-disk-icon,
[data-ext="jpeg"] .sb-disk-icon,
[data-ext="png"] .sb-disk-icon,
[data-ext="webp"] .sb-disk-icon {
    background: #fce7f3;
    color: #be185d;
}

/* =========================================================
   GRID / TILE VIEW
   ========================================================= */

.sb-disk-grid,
.sb-disk-tiles,
.sb-disk-tile-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(170px, 1fr));
    gap: 12px;
    margin-top: 14px;
}

.sb-disk-card,
.sb-disk-tile,
.sb-disk-grid-item {
    min-height: 150px;
    padding: 14px;
    border: 1px solid var(--disk-border);
    border-radius: 16px;
    background: #fff;
    cursor: pointer;
    transition:
        border-color .15s ease,
        box-shadow .15s ease,
        transform .15s ease,
        background .15s ease;
}

.sb-disk-card:hover,
.sb-disk-tile:hover,
.sb-disk-grid-item:hover {
    border-color: #c7d2fe;
    background: #f8fbff;
    box-shadow: 0 12px 28px rgba(37, 99, 235, .10);
    transform: translateY(-1px);
}

.sb-disk-card-preview,
.sb-disk-tile-preview {
    height: 82px;
    border-radius: 14px;
    background: #f1f5f9;
    display: flex;
    align-items: center;
    justify-content: center;
    margin-bottom: 12px;
}

.sb-disk-card-title,
.sb-disk-tile-title {
    font-size: 13px;
    line-height: 1.3;
    font-weight: 800;
    color: #111827;
    word-break: break-word;
}

.sb-disk-card-meta,
.sb-disk-tile-meta {
    margin-top: 4px;
    font-size: 12px;
    color: var(--disk-muted);
}

/* =========================================================
   EMPTY STATE
   ========================================================= */

.sb-public-disk-loading,
.sb-disk-empty,
.sb-disk-empty-state,
.sb-disk .empty,
.sb-disk [data-role="empty"] {
    min-height: 150px;
    padding: 28px;
    border: 1px dashed #cbd5e1;
    border-radius: 16px;
    background:
        radial-gradient(circle at top left, rgba(37, 99, 235, .08), transparent 35%),
        #ffffff;
    color: var(--disk-muted);
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    gap: 8px;
}

.sb-public-disk-loading::before,
.sb-disk-empty::before,
.sb-disk-empty-state::before,
.sb-disk .empty::before,
.sb-disk [data-role="empty"]::before {
    content: "📁";
    width: 54px;
    height: 54px;
    border-radius: 18px;
    background: #eef2ff;
    color: var(--disk-accent);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 26px;
    margin-bottom: 4px;
}

.sb-disk-empty strong,
.sb-disk-empty-state strong {
    color: var(--disk-text);
    font-size: 16px;
}

/* =========================================================
   MODALS / FORMS INSIDE DISK
   ========================================================= */

.sb-disk-modal,
.sb-disk-dialog {
    border-radius: 18px;
}

.sb-disk input[type="file"] {
    font-size: 13px;
}

/* =========================================================
   SMALL FIXES
   ========================================================= */

.sb-disk a {
    color: var(--disk-accent);
    text-decoration: none;
}

.sb-disk a:hover {
    text-decoration: underline;
}

.sb-disk * {
    box-sizing: border-box;
}

/* Убираем слишком тонкий/технический вид старых линий */
.sb-disk hr {
    border: 0;
    border-top: 1px solid var(--disk-border);
    margin: 14px 0;
}

/* =========================================================
   ADAPTIVE
   ========================================================= */

@media (max-width: 900px) {
    .sb-public-disk-head {
        flex-direction: column;
        align-items: stretch;
    }

    .sb-disk-toolbar,
    .sb-disk-actions,
    .sb-disk-topbar,
    .sb-disk-controls,
    .sb-disk-filter,
    .sb-disk-filters,
    .sb-disk-search-row {
        align-items: stretch;
        flex-direction: column;
    }

    .sb-disk input[type="text"],
    .sb-disk input[type="search"],
    .sb-disk select {
        width: 100%;
        min-width: 0;
    }

    .sb-disk button,
    .sb-public-block--disk button {
        width: 100%;
    }

    .sb-disk-grid,
    .sb-disk-tiles,
    .sb-disk-tile-list {
        grid-template-columns: repeat(auto-fill, minmax(140px, 1fr));
    }
}

После этого в public_page.php обнови версию CSS для диска.

Найди:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css">

Замени на:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=2">

И обнови публичную страницу через Ctrl + F5.

Это первый шаг варианта 2: визуально диск станет чище. После проверки дальше сделаем второй шаг — иконки файлов/папок и красивую пустую заглушку уже через script.js, чтобы они точно отображались, а не только через CSS.