Да, выносим стили editor.php в отдельный файл.

Создай файл:

/local/sitebuilder/assets/admin/editor.css

И вставь туда:

.sb-editor-shell {
    display: grid;
    grid-template-columns: 320px minmax(0, 1fr) 360px;
    gap: 20px;
    align-items: start;
}

.sb-editor-col {
    min-width: 0;
}

.sb-editor-sticky {
    position: sticky;
    top: 16px;
}

.sb-editor-topline {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 16px;
    margin-bottom: 18px;
}

.sb-editor-topline-note {
    margin: 0;
    color: #6b7280;
    font-size: 14px;
    max-width: 860px;
    line-height: 1.5;
}

.sb-editor-topline-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    justify-content: flex-end;
}

.sb-editor-section-head {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    margin-bottom: 14px;
}

.sb-editor-section-head .sb-panel-title {
    margin: 0;
}

.sb-editor-create {
    padding-bottom: 14px;
    margin-bottom: 14px;
    border-bottom: 1px solid #eef2f7;
}

.sb-editor-pages {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.sb-editor-page-item {
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    background: #fafafa;
    padding: 12px;
    cursor: pointer;
    transition: border-color .15s ease, background .15s ease, box-shadow .15s ease;
}

.sb-editor-page-item:hover {
    border-color: #c7d2fe;
    background: #fcfcff;
}

.sb-editor-page-item.is-active {
    border-color: #2563eb;
    background: #eff6ff;
    box-shadow: 0 6px 18px rgba(37, 99, 235, 0.12);
}

.sb-editor-page-top {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 12px;
}

.sb-editor-page-title {
    margin: 0 0 6px;
    font-size: 15px;
    font-weight: 700;
    color: #111827;
    line-height: 1.2;
}

.sb-editor-page-meta {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 8px;
}

.sb-editor-chip {
    display: inline-flex;
    align-items: center;
    min-height: 24px;
    padding: 0 8px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #4b5563;
    font-size: 12px;
    font-weight: 600;
}

.sb-editor-chip--blue {
    background: #eef2ff;
    color: #3730a3;
}

.sb-editor-chip--green {
    background: #dcfce7;
    color: #166534;
}

.sb-editor-chip--yellow {
    background: #fef3c7;
    color: #92400e;
}

.sb-editor-page-actions,
.sb-editor-block-actions,
.sb-editor-inspector-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 12px;
}

.sb-editor-page-actions .sb-btn,
.sb-editor-block-actions .sb-btn,
.sb-editor-inspector-actions .sb-btn {
    height: 32px;
    padding: 0 10px;
    font-size: 12px;
}

.sb-editor-canvas {
    background: #ffffff;
    border: 1px solid #e5e7eb;
    border-radius: 18px;
    box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
    overflow: hidden;
}

.sb-editor-canvas-head {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 12px;
    padding: 14px 18px;
    border-bottom: 1px solid #e5e7eb;
    background: #f9fafb;
}

.sb-editor-canvas-title {
    margin: 0;
    font-size: 18px;
    font-weight: 700;
    color: #111827;
}

.sb-editor-canvas-sub {
    margin: 4px 0 0;
    font-size: 13px;
    color: #6b7280;
}

.sb-editor-canvas-body {
    background: #f8fafc;
    padding: 24px;
    min-height: 720px;
}

.sb-editor-page {
    max-width: 980px;
    margin: 0 auto;
    background: #fff;
    border: 1px solid #e5e7eb;
    border-radius: 20px;
    min-height: 620px;
    padding: 24px;
    box-shadow: 0 12px 28px rgba(15, 23, 42, 0.06);
}

.sb-editor-page-heading {
    margin: 0 0 18px;
    font-size: 30px;
    line-height: 1.15;
    font-weight: 700;
    color: #111827;
}

.sb-editor-addbar {
    display: grid;
    grid-template-columns: repeat(5, minmax(0, 1fr));
    gap: 10px;
    margin-bottom: 18px;
}

.sb-editor-add-card {
    border: 1px solid #dbe3f0;
    border-radius: 14px;
    background: #fff;
    padding: 12px;
    text-align: left;
    cursor: pointer;
    transition: border-color .15s ease, transform .15s ease, box-shadow .15s ease;
}

.sb-editor-add-card:hover {
    border-color: #93c5fd;
    transform: translateY(-1px);
    box-shadow: 0 8px 18px rgba(37, 99, 235, 0.08);
}

.sb-editor-add-card__title {
    display: block;
    font-size: 14px;
    font-weight: 700;
    color: #111827;
    margin-bottom: 4px;
}

.sb-editor-add-card__text {
    display: block;
    font-size: 12px;
    color: #6b7280;
    line-height: 1.4;
}

.sb-editor-blocks {
    display: flex;
    flex-direction: column;
    gap: 14px;
}

.sb-editor-block {
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    background: #fff;
    padding: 14px;
    transition: border-color .15s ease, box-shadow .15s ease, background .15s ease;
    cursor: pointer;
}

.sb-editor-block:hover {
    border-color: #c7d2fe;
    box-shadow: 0 8px 20px rgba(37, 99, 235, 0.08);
}

.sb-editor-block.is-active {
    border-color: #2563eb;
    background: #f8fbff;
    box-shadow: 0 10px 24px rgba(37, 99, 235, 0.12);
}

.sb-editor-block-head {
    display: flex;
    justify-content: space-between;
    align-items: flex-start;
    gap: 12px;
    margin-bottom: 10px;
}

.sb-editor-block-title {
    margin: 0;
    font-size: 14px;
    font-weight: 700;
    color: #111827;
}

.sb-editor-block-preview {
    border: 1px solid #eef2f7;
    background: #fff;
    border-radius: 12px;
    padding: 12px;
    font-size: 14px;
    line-height: 1.6;
    color: #374151;
    min-height: 52px;
}

.sb-editor-empty-big {
    padding: 30px 20px;
    text-align: center;
    color: #6b7280;
    border: 1px dashed #d1d5db;
    border-radius: 16px;
    background: #fff;
}

.sb-editor-empty-big strong {
    display: block;
    color: #111827;
    margin-bottom: 6px;
    font-size: 16px;
}

.sb-editor-note {
    font-size: 13px;
    color: #6b7280;
    line-height: 1.5;
    margin-top: -2px;
    margin-bottom: 12px;
}

.sb-editor-json-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 12px;
}

.sb-editor-divider {
    height: 1px;
    background: #eef2f7;
    margin: 14px 0;
}

.sb-block-type-form {
    display: none;
}

.sb-block-type-form.is-active {
    display: block;
}

.sb-editor-advanced-json {
    display: none;
}

.sb-editor-advanced-json.is-open {
    display: block;
}

.sb-block-form-note {
    margin: 6px 0 0;
    font-size: 12px;
    color: #6b7280;
    line-height: 1.4;
}

/* Поиск пользователя в правах */
.sb-access-search-wrap {
    position: relative;
}

.sb-access-results {
    display: none;
    position: absolute;
    z-index: 1000;
    left: 0;
    right: 0;
    top: calc(100% + 6px);
    max-height: 260px;
    overflow: auto;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    background: #fff;
    box-shadow: 0 16px 40px rgba(15, 23, 42, 0.16);
}

.sb-access-results.is-open {
    display: block;
}

.sb-access-result-item {
    width: 100%;
    display: block;
    text-align: left;
    padding: 10px 12px;
    border: 0;
    background: #fff;
    cursor: pointer;
    border-bottom: 1px solid #f3f4f6;
}

.sb-access-result-item:hover {
    background: #f8fafc;
}

.sb-access-result-title {
    font-size: 13px;
    font-weight: 700;
    color: #111827;
}

.sb-access-result-meta {
    margin-top: 3px;
    font-size: 12px;
    color: #6b7280;
}

.sb-access-selected {
    margin-top: 8px;
    padding: 9px 10px;
    border-radius: 12px;
    border: 1px solid #bbf7d0;
    background: #f0fdf4;
    color: #166534;
    font-size: 13px;
    line-height: 1.4;
}

.sb-access-selected button {
    margin-left: 8px;
}

/* Форма прав в узкой колонке */
.sb-access-form {
    display: grid;
    grid-template-columns: 1fr !important;
    gap: 12px;
}

.sb-access-form .sb-field {
    min-width: 0;
}

.sb-access-form .sb-input,
.sb-access-form .sb-select {
    width: 100%;
    max-width: 100%;
    box-sizing: border-box;
}

.sb-access-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-access-item {
    display: grid;
    grid-template-columns: minmax(0, 1fr) auto auto;
    gap: 8px;
    align-items: center;
    max-width: 100%;
    overflow: hidden;
}

.sb-access-item__main {
    min-width: 0;
    overflow: hidden;
}

.sb-access-item__name,
.sb-access-item__meta {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-item .sb-role-badge {
    white-space: nowrap;
}

.sb-access-item .sb-btn {
    white-space: nowrap;
}

@media (max-width: 1500px) {
    .sb-access-item {
        grid-template-columns: minmax(0, 1fr) auto;
    }

    .sb-access-item .sb-btn-danger {
        grid-column: 1 / -1;
        justify-self: flex-start;
    }
}

@media (max-width: 1440px) {
    .sb-editor-shell {
        grid-template-columns: 300px minmax(0, 1fr) 330px;
    }

    .sb-editor-addbar {
        grid-template-columns: repeat(2, minmax(0, 1fr));
    }
}

@media (max-width: 1180px) {
    .sb-editor-shell {
        grid-template-columns: 320px minmax(0, 1fr);
    }

    .sb-editor-col--right {
        grid-column: 1 / -1;
    }

    .sb-editor-sticky {
        position: static;
    }
}

@media (max-width: 900px) {
    .sb-editor-topline {
        flex-direction: column;
        align-items: stretch;
    }

    .sb-editor-topline-actions {
        justify-content: flex-start;
    }

    .sb-editor-shell {
        grid-template-columns: 1fr;
    }

    .sb-editor-addbar {
        grid-template-columns: 1fr;
    }
}

Теперь в editor.php в <head> добавь подключение:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.css">

Должно быть примерно так:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.css">

После этого из editor.php можно удалить весь большой блок:

<style>
    ...
</style>

То есть в editor.php оставляем только подключение CSS-файла.