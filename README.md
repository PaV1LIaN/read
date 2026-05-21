Да, тут лучше не трогать рабочий script.js. Сделаем через CSS:

1. скрываем дублирующий заголовок Файлы;


2. скрываем строку 4 файлов · 0 папок;


3. уменьшаем лишние отступы;


4. приводим окно настроек диска к нормальному виду.



В конец файла:

/local/sitebuilder/components/disk/styles.css

добавь этот блок:

/* =========================================================
   DISK COMPACT FIX + NICE SETTINGS MODAL
   ========================================================= */

/* Убираем дублирующий заголовок "Файлы" и количество */
.sb-disk--modern .sb-disk__title,
.sb-disk--modern .sb-disk-title,
.sb-disk--modern [data-role="subtitle"],
.sb-disk--modern .sb-disk__subtitle,
.sb-disk--modern .sb-disk-subtitle {
    display: none !important;
}

/* Уменьшаем лишнее пространство сверху внутри диска */
.sb-disk--modern .sb-disk__top,
.sb-disk--modern .sb-disk__header {
    margin-bottom: 8px !important;
}

.sb-disk--modern .sb-disk__toolbar,
.sb-disk--modern .sb-disk__controls,
.sb-disk--modern .sb-disk__actions-panel,
.sb-disk--modern .sb-disk__filter,
.sb-disk--modern .sb-disk__filters {
    margin: 6px 0 10px !important;
    padding: 10px !important;
    border-radius: 14px !important;
}

/* Если есть блок заголовка диска, делаем его компактным */
.sb-disk--modern .sb-disk__head,
.sb-disk--modern .sb-disk-head {
    min-height: 0 !important;
    margin: 0 0 6px !important;
    padding: 0 !important;
}

/* Панель поиска и кнопок компактнее */
.sb-disk--modern input[type="text"],
.sb-disk--modern input[type="search"],
.sb-disk--modern select {
    height: 36px !important;
    border-radius: 10px !important;
}

.sb-disk--modern input[type="text"],
.sb-disk--modern input[type="search"] {
    min-width: 240px !important;
    max-width: 360px !important;
}

.sb-disk--modern button,
.sb-disk--modern .sb-disk__row-btn,
.sb-disk--modern .sb-disk__view-btn,
.sb-disk--modern .sb-disk-modern-control {
    min-height: 34px !important;
    padding: 0 11px !important;
    border-radius: 10px !important;
}

/* Таблица ближе к панели */
.sb-disk--modern [data-view-container="table"],
.sb-disk--modern .sb-disk__table-wrap,
.sb-disk--modern .sb-disk__table-container {
    margin-top: 8px !important;
}

.sb-disk--modern thead th {
    height: 38px !important;
}

.sb-disk--modern tbody td {
    height: 50px !important;
    padding-top: 8px !important;
    padding-bottom: 8px !important;
}

/* =========================================================
   SETTINGS MODAL — красивое окно настроек
   ========================================================= */

.sb-disk [data-role="settings-modal"] {
    position: fixed !important;
    inset: 0 !important;
    z-index: 10000 !important;
    display: flex !important;
    align-items: center !important;
    justify-content: center !important;
    padding: 24px !important;
    background: rgba(15, 23, 42, .45) !important;
    backdrop-filter: blur(6px);
    overflow: auto !important;
}

.sb-disk [data-role="settings-modal"][hidden] {
    display: none !important;
}

.sb-disk [data-role="settings-modal"] > * {
    width: min(760px, 100%) !important;
    max-height: calc(100vh - 48px) !important;
    overflow: auto !important;
    margin: 0 !important;
    padding: 22px !important;
    border: 1px solid rgba(229, 231, 235, .9) !important;
    border-radius: 22px !important;
    background: #ffffff !important;
    box-shadow: 0 30px 90px rgba(15, 23, 42, .28) !important;
}

/* Заголовок настроек */
.sb-disk [data-role="settings-modal"] h1,
.sb-disk [data-role="settings-modal"] h2,
.sb-disk [data-role="settings-modal"] h3 {
    margin: 0 0 18px !important;
    color: #111827 !important;
    font-size: 22px !important;
    line-height: 1.25 !important;
    font-weight: 900 !important;
}

/* Кнопка закрытия, если она маленькая */
.sb-disk [data-role="settings-modal"] [data-action="close-settings"] {
    min-width: 34px !important;
    width: 34px !important;
    height: 34px !important;
    min-height: 34px !important;
    padding: 0 !important;
    border-radius: 12px !important;
    border: 1px solid #e5e7eb !important;
    background: #f8fafc !important;
    color: #374151 !important;
}

/* Форма настроек */
.sb-disk [data-role="settings-form"] {
    display: grid !important;
    grid-template-columns: 190px minmax(0, 1fr) !important;
    gap: 12px 14px !important;
    align-items: center !important;
}

/* Если label и input идут отдельными элементами */
.sb-disk [data-role="settings-form"] label {
    margin: 0 !important;
    color: #374151 !important;
    font-size: 13px !important;
    font-weight: 700 !important;
}

/* Поля */
.sb-disk [data-role="settings-form"] input[type="text"],
.sb-disk [data-role="settings-form"] input[type="number"],
.sb-disk [data-role="settings-form"] select,
.sb-disk [data-role="settings-form"] textarea {
    width: 100% !important;
    min-width: 0 !important;
    max-width: 100% !important;
    height: 38px !important;
    padding: 0 12px !important;
    border: 1px solid #dbe3ef !important;
    border-radius: 12px !important;
    background: #fff !important;
    color: #111827 !important;
    font-size: 13px !important;
    outline: none !important;
}

.sb-disk [data-role="settings-form"] textarea {
    height: auto !important;
    min-height: 76px !important;
    padding-top: 10px !important;
    padding-bottom: 10px !important;
}

.sb-disk [data-role="settings-form"] input:focus,
.sb-disk [data-role="settings-form"] select:focus,
.sb-disk [data-role="settings-form"] textarea:focus {
    border-color: var(--disk-accent, #2563eb) !important;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12) !important;
}

/* Подсказка про расширения */
.sb-disk [data-role="settings-form"] small,
.sb-disk [data-role="settings-form"] .hint,
.sb-disk [data-role="settings-form"] .help {
    grid-column: 2 / 3 !important;
    margin-top: -6px !important;
    color: #6b7280 !important;
    font-size: 12px !important;
}

/* Чекбоксы в аккуратную сетку */
.sb-disk [data-role="settings-form"] input[type="checkbox"] {
    width: 16px !important;
    height: 16px !important;
    margin: 0 6px 0 0 !important;
    accent-color: var(--disk-accent, #2563eb);
}

.sb-disk [data-role="settings-form"] label:has(input[type="checkbox"]) {
    grid-column: span 1 !important;
    display: inline-flex !important;
    align-items: center !important;
    min-height: 30px !important;
    padding: 6px 8px !important;
    border: 1px solid #e5e7eb !important;
    border-radius: 12px !important;
    background: #f8fafc !important;
    color: #374151 !important;
    font-size: 12px !important;
}

/* Кнопки внизу модалки */
.sb-disk [data-role="settings-modal"] [data-action="save-settings"] {
    border-color: var(--disk-accent, #2563eb) !important;
    background: var(--disk-accent, #2563eb) !important;
    color: #fff !important;
}

.sb-disk [data-role="settings-modal"] [data-action="save-settings"]:hover {
    color: #fff !important;
    box-shadow: 0 8px 20px rgba(37, 99, 235, .22) !important;
}

.sb-disk [data-role="settings-message"] {
    margin-top: 12px !important;
    color: #6b7280 !important;
    font-size: 13px !important;
}

/* Адаптив модалки */
@media (max-width: 760px) {
    .sb-disk [data-role="settings-modal"] {
        align-items: flex-start !important;
        padding: 12px !important;
    }

    .sb-disk [data-role="settings-modal"] > * {
        padding: 16px !important;
        border-radius: 18px !important;
        max-height: none !important;
    }

    .sb-disk [data-role="settings-form"] {
        grid-template-columns: 1fr !important;
        gap: 8px !important;
    }

    .sb-disk [data-role="settings-form"] small,
    .sb-disk [data-role="settings-form"] .hint,
    .sb-disk [data-role="settings-form"] .help {
        grid-column: auto !important;
    }

    .sb-disk [data-role="settings-form"] label:has(input[type="checkbox"]) {
        grid-column: auto !important;
    }
}

Потом в public_page.php обнови версию:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=8">

После Ctrl + F5 должно стать компактнее: заголовок Файлы и счётчик исчезнут, таблица подтянется выше, а окно настроек станет нормальной модалкой с аккуратными полями.