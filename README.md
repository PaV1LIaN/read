Не заменяй весь public.css, иначе опять можно снести рабочий визуал. Просто добавь этот блок в самый конец файла:

/* =========================================================
   SITEBUILDER PAGE SECTIONS FIX
   Секции и колонки на публичной странице
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

/* Новый HTML секций */
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

/* Старый HTML секций, если где-то остался */
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

/* Чтобы блоки внутри колонок не ломали ширину */
.sb-page-section__column > *,
.sb-section-column > * {
    max-width: 100%;
    min-width: 0;
    box-sizing: border-box;
}

/* Отступы блоков внутри колонок */
.sb-page-section__column > * + *,
.sb-section-column > * + * {
    margin-top: 16px;
}

/* На телефонах все колонки становятся одной */
@media (max-width: 900px) {
    .sb-page-section__grid,
    .sb-section-grid {
        grid-template-columns: 1fr !important;
    }
}

И в public_page.php обнови версию подключения CSS:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=8">

После этого на публичной странице сделай Ctrl + F5.