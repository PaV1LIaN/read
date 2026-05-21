Заменяй полностью файл:

/local/sitebuilder/components/disk/styles.css

на этот:

/* =========================================================
   SITEBUILDER DISK
   Modern File Manager UI
   ========================================================= */

.sb-disk {
    --disk-accent: var(--sb-accent, #2563eb);
    --disk-accent-soft: rgba(37, 99, 235, .08);
    --disk-bg: #ffffff;
    --disk-soft: #f8fafc;
    --disk-soft-2: #f1f5f9;
    --disk-border: #e5e7eb;
    --disk-border-2: #dbe3ef;
    --disk-text: #111827;
    --disk-muted: #6b7280;
    --disk-danger: #dc2626;
    --disk-radius: 18px;
    --disk-shadow: 0 14px 34px rgba(15, 23, 42, .06);

    width: 100%;
    min-width: 0;
    box-sizing: border-box;
    color: var(--disk-text);
}

.sb-disk *,
.sb-disk *::before,
.sb-disk *::after {
    box-sizing: border-box;
}

.sb-disk--modern {
    width: 100%;
    min-width: 0;
}

/* =========================================================
   COMMON
   ========================================================= */

.sb-disk a {
    color: var(--disk-accent);
    text-decoration: none;
}

.sb-disk a:hover {
    text-decoration: underline;
}

.sb-disk button,
.sb-disk input,
.sb-disk select,
.sb-disk textarea {
    font-family: inherit;
}

.sb-disk [hidden] {
    display: none !important;
}

/* =========================================================
   BLOCK HEAD
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
.sb-disk__title,
.sb-disk-title,
.sb-disk h2,
.sb-disk h3 {
    margin: 0;
    font-size: 22px;
    line-height: 1.25;
    font-weight: 800;
    color: var(--disk-text);
}

.sb-disk__subtitle,
.sb-disk-subtitle,
.sb-disk__meta,
.sb-disk-meta,
.sb-disk__description {
    margin-top: 4px;
    color: var(--disk-muted);
    font-size: 13px;
    line-height: 1.45;
}

/* =========================================================
   TOP / TOOLBAR / CONTROLS
   ========================================================= */

.sb-disk__top,
.sb-disk__header,
.sb-disk__toolbar,
.sb-disk__controls,
.sb-disk__actions-panel,
.sb-disk__panel,
.sb-disk__filter,
.sb-disk__filters {
    display: flex;
    align-items: center;
    gap: 10px;
    flex-wrap: wrap;
    width: 100%;
    min-width: 0;
}

.sb-disk__top,
.sb-disk__header {
    justify-content: space-between;
    margin-bottom: 14px;
}

.sb-disk__toolbar,
.sb-disk__controls,
.sb-disk__actions-panel,
.sb-disk__filter,
.sb-disk__filters {
    margin: 12px 0;
    padding: 14px;
    border: 1px solid var(--disk-border);
    border-radius: var(--disk-radius);
    background: var(--disk-soft);
}

.sb-disk--modern [data-role="search-input"],
.sb-disk--modern [data-role="sort-select"],
.sb-disk--modern [data-action="upload"],
.sb-disk--modern [data-action="create-folder"],
.sb-disk--modern [data-action="refresh"],
.sb-disk--modern [data-action="settings"],
.sb-disk--modern .sb-disk__view-btn {
    margin: 0;
}

/* Search / selects */

.sb-disk--modern input[type="text"],
.sb-disk--modern input[type="search"],
.sb-disk--modern select {
    height: 40px;
    border: 1px solid var(--disk-border-2);
    border-radius: 12px;
    background: #fff;
    color: var(--disk-text);
    font-size: 13px;
    outline: none;
    transition: border-color .15s ease, box-shadow .15s ease;
}

.sb-disk--modern input[type="text"],
.sb-disk--modern input[type="search"] {
    min-width: 280px;
    max-width: 460px;
    flex: 1 1 280px;
    padding: 0 14px;
}

.sb-disk--modern select {
    min-width: 170px;
    flex: 0 0 auto;
    padding: 0 12px;
}

.sb-disk--modern input[type="text"]:focus,
.sb-disk--modern input[type="search"]:focus,
.sb-disk--modern select:focus {
    border-color: var(--disk-accent);
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12);
}

/* Buttons */

.sb-disk--modern button,
.sb-disk--modern .sb-disk__row-btn,
.sb-disk--modern .sb-disk__view-btn,
.sb-disk--modern .sb-disk-modern-control {
    min-height: 38px;
    padding: 0 13px;
    border-radius: 12px;
    border: 1px solid var(--disk-border-2);
    background: #fff;
    color: #374151;
    font-size: 13px;
    font-weight: 700;
    line-height: 1;
    cursor: pointer;
    white-space: nowrap;
    transition:
        background .15s ease,
        border-color .15s ease,
        color .15s ease,
        box-shadow .15s ease,
        transform .15s ease;
}

.sb-disk--modern button:hover,
.sb-disk--modern .sb-disk__row-btn:hover,
.sb-disk--modern .sb-disk__view-btn:hover {
    border-color: #c7d2fe;
    background: #eef2ff;
    color: var(--disk-accent);
}

.sb-disk--modern button:active,
.sb-disk--modern .sb-disk__row-btn:active,
.sb-disk--modern .sb-disk__view-btn:active {
    transform: translateY(1px);
}

.sb-disk--modern [data-action="upload"],
.sb-disk--modern .sb-disk-modern-primary,
.sb-disk--modern .sb-disk__row-btn.is-primary {
    border-color: var(--disk-accent);
    background: var(--disk-accent);
    color: #fff;
}

.sb-disk--modern [data-action="upload"]:hover,
.sb-disk--modern .sb-disk-modern-primary:hover,
.sb-disk--modern .sb-disk__row-btn.is-primary:hover {
    border-color: var(--disk-accent);
    background: var(--disk-accent);
    color: #fff;
    box-shadow: 0 8px 20px rgba(37, 99, 235, .22);
}

.sb-disk--modern .sb-disk__view-btn.is-active {
    border-color: #c7d2fe;
    background: #eef2ff;
    color: var(--disk-accent);
}

/* Danger */

.sb-disk--modern .sb-disk__row-btn.is-danger,
.sb-disk--modern [data-row-action="delete"] {
    border-color: #fecaca;
    background: #fff;
    color: var(--disk-danger);
}

.sb-disk--modern .sb-disk__row-btn.is-danger:hover,
.sb-disk--modern [data-row-action="delete"]:hover {
    background: #fee2e2;
    color: #991b1b;
}

/* =========================================================
   BREADCRUMBS
   ========================================================= */

.sb-disk__breadcrumbs,
.sb-disk [data-role="breadcrumbs"] {
    display: flex;
    align-items: center;
    flex-wrap: wrap;
    gap: 6px;
    margin: 10px 0 14px;
    color: var(--disk-muted);
    font-size: 13px;
}

.sb-disk__crumb {
    display: inline-flex;
    align-items: center;
    min-height: 30px;
    padding: 0 10px;
    border: 1px solid transparent !important;
    border-radius: 999px !important;
    background: #f3f4f6 !important;
    color: #374151 !important;
    font-size: 13px !important;
    font-weight: 700 !important;
}

.sb-disk__crumb:hover {
    background: #eef2ff !important;
    color: var(--disk-accent) !important;
}

.sb-disk__crumb-separator,
.sb-disk [data-role="breadcrumbs"] > span {
    color: #9ca3af;
}

/* =========================================================
   STATES
   ========================================================= */

.sb-disk [data-state] {
    margin-top: 14px;
}

.sb-disk [data-state="loading"],
.sb-disk [data-state="error"],
.sb-disk [data-state="no-root"],
.sb-disk [data-state="no-access"],
.sb-disk [data-state="empty"],
.sb-public-disk-loading {
    min-height: 160px;
    padding: 30px;
    border: 1px dashed #cbd5e1;
    border-radius: 18px;
    background:
        radial-gradient(circle at top left, rgba(37, 99, 235, .08), transparent 35%),
        #fff;
    color: var(--disk-muted);
    text-align: center;
    display: flex;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    gap: 8px;
}

.sb-disk [data-state="error"] {
    border-color: #fecaca;
    background:
        radial-gradient(circle at top left, rgba(220, 38, 38, .08), transparent 35%),
        #fff;
    color: #991b1b;
}

.sb-disk-empty-enhanced {
    min-height: 160px;
    padding: 30px;
    border: 1px dashed #cbd5e1;
    border-radius: 18px;
    background:
        radial-gradient(circle at top left, rgba(37, 99, 235, .08), transparent 35%),
        #fff;
    color: var(--disk-muted);
    text-align: center;
    display: flex !important;
    align-items: center;
    justify-content: center;
    flex-direction: column;
    gap: 8px;
}

.sb-disk-empty-icon {
    width: 58px;
    height: 58px;
    border-radius: 20px;
    background: #eef2ff;
    color: var(--disk-accent);
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 30px;
}

.sb-disk-empty-enhanced strong,
.sb-disk [data-state] strong {
    color: var(--disk-text);
    font-size: 16px;
}

.sb-disk-empty-enhanced span,
.sb-disk [data-state] span {
    color: var(--disk-muted);
    font-size: 13px;
}

/* =========================================================
   TABLE VIEW
   ========================================================= */

.sb-disk__table-wrap,
.sb-disk__table-container,
.sb-disk [data-view-container="table"] {
    width: 100%;
    min-width: 0;
    overflow-x: auto;
    border: 1px solid var(--disk-border);
    border-radius: 16px;
    background: #fff;
}

.sb-disk--modern table {
    width: 100%;
    min-width: 760px;
    border-collapse: separate;
    border-spacing: 0;
    background: #fff;
}

.sb-disk--modern thead th {
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

.sb-disk--modern tbody td {
    height: 58px;
    padding: 10px 14px;
    border-bottom: 1px solid #f1f5f9;
    vertical-align: middle;
    color: #374151;
    font-size: 13px;
}

.sb-disk--modern tbody tr:last-child td {
    border-bottom: 0;
}

.sb-disk--modern tbody tr {
    transition: background .15s ease;
}

.sb-disk--modern tbody tr:hover {
    background: #f8fbff;
}

.sb-disk--modern tbody tr.is-selected {
    background: #eff6ff;
}

.sb-disk__check-cell {
    width: 44px;
}

.sb-disk__name-cell {
    min-width: 260px;
}

/* =========================================================
   MODERN NAME / ICONS
   ========================================================= */

.sb-disk__modern-name {
    display: flex;
    align-items: center;
    gap: 12px;
    min-width: 0;
    max-width: 100%;
}

.sb-disk__modern-name-main {
    min-width: 0;
}

.sb-disk__modern-name-title {
    color: var(--disk-text);
    font-size: 14px;
    font-weight: 800;
    line-height: 1.25;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.sb-disk__modern-name-sub {
    margin-top: 3px;
    color: var(--disk-muted);
    font-size: 12px;
}

.sb-disk__modern-icon {
    width: 38px;
    height: 38px;
    min-width: 38px;
    max-width: 38px;
    min-height: 38px;
    max-height: 38px;
    border-radius: 13px;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    background: #f1f5f9;
    color: #475569;
    font-size: 10px;
    font-weight: 900;
    line-height: 1;
    flex: 0 0 38px;
}

.sb-disk__modern-icon--folder {
    background: #fef3c7;
    color: #92400e;
    font-size: 20px;
}

.sb-disk__modern-icon--pdf {
    background: #fee2e2;
    color: #991b1b;
}

.sb-disk__modern-icon--doc {
    background: #dbeafe;
    color: #1d4ed8;
}

.sb-disk__modern-icon--xls {
    background: #dcfce7;
    color: #166534;
}

.sb-disk__modern-icon--ppt {
    background: #ffedd5;
    color: #c2410c;
}

.sb-disk__modern-icon--img {
    background: #fce7f3;
    color: #be185d;
}

.sb-disk__modern-icon--zip {
    background: #ede9fe;
    color: #6d28d9;
}

.sb-disk__modern-icon--file {
    background: #f1f5f9;
    color: #475569;
}

/* Старые бейджи и новые pill */

.sb-disk__badge,
.sb-disk__type-pill {
    display: inline-flex;
    align-items: center;
    min-height: 24px;
    padding: 0 9px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #4b5563;
    font-size: 12px;
    font-weight: 800;
    white-space: nowrap;
}

/* =========================================================
   ROW ACTIONS
   ========================================================= */

.sb-disk--modern .sb-disk__actions {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 6px;
    flex-wrap: wrap;
}

.sb-disk--modern .sb-disk__actions .sb-disk__row-btn {
    min-height: 32px;
    padding: 0 10px;
    border-radius: 10px;
    font-size: 12px;
}

/* =========================================================
   GRID VIEW
   ========================================================= */

.sb-disk--modern .sb-disk__grid,
.sb-disk--modern [data-view-container="grid"] {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(180px, 1fr));
    gap: 12px;
    width: 100%;
    min-width: 0;
}

.sb-disk--modern .sb-disk__card {
    min-height: 220px;
    padding: 14px;
    border: 1px solid var(--disk-border);
    border-radius: 18px;
    background: #fff;
    transition:
        border-color .15s ease,
        box-shadow .15s ease,
        transform .15s ease,
        background .15s ease;
}

.sb-disk--modern .sb-disk__card:hover {
    border-color: #c7d2fe;
    background: #f8fbff;
    box-shadow: 0 12px 26px rgba(37, 99, 235, .10);
    transform: translateY(-1px);
}

.sb-disk--modern .sb-disk__card.is-selected {
    border-color: var(--disk-accent);
    background: #eff6ff;
}

.sb-disk--modern .sb-disk__card-top {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 8px;
}

.sb-disk__card-check {
    display: inline-flex;
    align-items: center;
}

.sb-disk--modern .sb-disk__card-preview {
    height: 86px;
    margin: 12px 0;
    border-radius: 15px;
    background: #f8fafc;
    display: flex;
    align-items: center;
    justify-content: center;
}

.sb-disk--modern .sb-disk__card-preview .sb-disk__modern-icon {
    width: 48px;
    height: 48px;
    min-width: 48px;
    min-height: 48px;
    border-radius: 16px;
}

.sb-disk--modern .sb-disk__card-preview .sb-disk__modern-icon--folder {
    font-size: 24px;
}

.sb-disk--modern .sb-disk__card-name {
    font-size: 14px;
    font-weight: 800;
    color: var(--disk-text);
    line-height: 1.3;
    word-break: break-word;
}

.sb-disk--modern .sb-disk__card-meta {
    margin-top: 5px;
    color: var(--disk-muted);
    font-size: 12px;
}

.sb-disk--modern .sb-disk__card-sub {
    color: var(--disk-muted);
}

.sb-disk--modern .sb-disk__card-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 12px;
}

.sb-disk--modern .sb-disk__card-actions .sb-disk__row-btn {
    min-height: 32px;
    padding: 0 10px;
    border-radius: 10px;
    font-size: 12px;
}

/* =========================================================
   BULK BAR
   ========================================================= */

.sb-disk [data-role="bulkbar"] {
    margin: 12px 0;
    padding: 12px 14px;
    border: 1px solid #c7d2fe;
    border-radius: 14px;
    background: #eef2ff;
    color: #1e3a8a;
}

.sb-disk [data-role="bulkbar-text"] {
    font-weight: 800;
}

/* =========================================================
   SETTINGS MODAL
   ========================================================= */

.sb-disk [data-role="settings-modal"] {
    position: fixed;
    inset: 0;
    z-index: 1000;
    background: rgba(15, 23, 42, .38);
    padding: 24px;
    overflow: auto;
}

.sb-disk [data-role="settings-modal"] > * {
    max-width: 780px;
    margin: 40px auto;
    border-radius: 20px;
    background: #fff;
    box-shadow: 0 24px 70px rgba(15, 23, 42, .22);
}

.sb-disk [data-role="settings-form"] {
    display: grid;
    gap: 12px;
}

.sb-disk [data-role="settings-message"] {
    color: var(--disk-muted);
    font-size: 13px;
}

/* =========================================================
   VIEWER BUTTON SPAN FIX
   ========================================================= */

.sb-disk__viewer-btn {
    display: inline-flex;
    align-items: center;
    justify-content: center;
    min-height: 32px;
    padding: 0 10px;
    border-radius: 10px;
    border: 1px solid var(--disk-accent);
    background: var(--disk-accent);
    color: #fff;
    font-size: 12px;
    font-weight: 700;
    cursor: pointer;
}

.sb-disk__viewer-btn:hover {
    color: #fff;
    box-shadow: 0 8px 18px rgba(37, 99, 235, .20);
}

/* =========================================================
   OLD NAME COMPATIBILITY
   ========================================================= */

.sb-disk__item-name {
    display: flex;
    align-items: center;
    gap: 8px;
    min-width: 0;
}

.sb-disk__item-name-label {
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

/* =========================================================
   RESPONSIVE
   ========================================================= */

@media (max-width: 1100px) {
    .sb-disk__top,
    .sb-disk__header {
        align-items: flex-start;
        flex-direction: column;
    }

    .sb-disk__toolbar,
    .sb-disk__controls,
    .sb-disk__actions-panel,
    .sb-disk__filter,
    .sb-disk__filters {
        align-items: stretch;
        flex-direction: column;
    }

    .sb-disk--modern input[type="text"],
    .sb-disk--modern input[type="search"],
    .sb-disk--modern select {
        width: 100%;
        max-width: 100%;
        min-width: 0;
    }

    .sb-disk--modern .sb-disk__actions {
        justify-content: flex-start;
    }
}

@media (max-width: 900px) {
    .sb-public-disk-head {
        flex-direction: column;
        align-items: stretch;
    }

    .sb-disk--modern .sb-disk__grid,
    .sb-disk--modern [data-view-container="grid"] {
        grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    }

    .sb-disk--modern table {
        min-width: 700px;
    }
}

@media (max-width: 700px) {
    .sb-disk__toolbar,
    .sb-disk__controls,
    .sb-disk__actions-panel,
    .sb-disk__filter,
    .sb-disk__filters {
        padding: 12px;
    }

    .sb-disk--modern button,
    .sb-disk--modern .sb-disk__view-btn,
    .sb-disk--modern [data-action="upload"],
    .sb-disk--modern [data-action="create-folder"],
    .sb-disk--modern [data-action="refresh"],
    .sb-disk--modern [data-action="settings"] {
        width: 100%;
    }

    .sb-disk--modern .sb-disk__actions,
    .sb-disk--modern .sb-disk__card-actions {
        align-items: stretch;
        flex-direction: column;
    }

    .sb-disk--modern .sb-disk__actions .sb-disk__row-btn,
    .sb-disk--modern .sb-disk__card-actions .sb-disk__row-btn {
        width: 100%;
    }
}

В public_page.php подключение сделай так:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=7">

После замены обнови страницу через Ctrl + F5.