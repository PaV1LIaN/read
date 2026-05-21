Да, это из-за последнего CSS:

.sb-public-header .sb-container {
    display: flex;
    justify-content: space-between;
}

Он разнёс бренд и меню по разным сторонам. Нам нужно сделать шапку так:

[логотип + название сайта]
[Домашняя] [Диск]

1. В public_page.php замени блок шапки

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди:

<?php if ($vm['showHeader']): ?>
    <header class="sb-public-header">
        <div class="sb-container">
            <?php if ($headerHtml !== ''): ?>
                <?= $headerHtml ?>
            <?php else: ?>
                <div class="sb-brand">
                    <?= sb_public_appearance_brand($site, $appearance) ?>
                </div>
            <?php endif; ?>

            <?php if ($menuHtml !== ''): ?>
                <?= $menuHtml ?>
            <?php endif; ?>
        </div>
    </header>
<?php endif; ?>

Замени на:

<?php if ($vm['showHeader']): ?>
    <header class="sb-public-header">
        <div class="sb-container sb-header-container">
            <div class="sb-header-brand-row">
                <div class="sb-brand">
                    <?= sb_public_appearance_brand($site, $appearance) ?>
                </div>

                <?php if ($headerHtml !== ''): ?>
                    <div class="sb-header-custom">
                        <?= $headerHtml ?>
                    </div>
                <?php endif; ?>
            </div>

            <?php if ($menuHtml !== ''): ?>
                <div class="sb-header-menu-row">
                    <?= $menuHtml ?>
                </div>
            <?php endif; ?>
        </div>
    </header>
<?php endif; ?>


---

2. В конец public.css добавь

Файл:

/local/sitebuilder/assets/public/public.css

/* =========================================================
   HEADER FIX: бренд сверху, меню под ним
   ========================================================= */

.sb-public-header .sb-container.sb-header-container {
    display: flex !important;
    flex-direction: column !important;
    align-items: flex-start !important;
    justify-content: flex-start !important;
    gap: 10px !important;
    min-width: 0 !important;
}

.sb-header-brand-row {
    width: 100%;
    min-width: 0;
    display: flex;
    align-items: center;
    justify-content: flex-start;
    gap: 16px;
}

.sb-header-custom {
    min-width: 0;
    flex: 1 1 auto;
}

.sb-header-menu-row {
    width: 100%;
    min-width: 0;
    display: flex;
    justify-content: flex-start;
}

.sb-header-menu-row .sb-public-menu {
    justify-content: flex-start !important;
    align-items: center;
    width: auto;
    max-width: 100%;
}

.sb-brand {
    display: inline-flex !important;
    align-items: center !important;
    justify-content: flex-start !important;
    gap: 10px !important;
    min-width: 0 !important;
    max-width: 100% !important;
    flex: 0 1 auto !important;
    flex-wrap: nowrap !important;
}

.sb-brand__logo {
    width: var(--sb-logo-size, 42px) !important;
    height: var(--sb-logo-size, 42px) !important;
    min-width: var(--sb-logo-size, 42px) !important;
    max-width: var(--sb-logo-size, 42px) !important;
    min-height: var(--sb-logo-size, 42px) !important;
    max-height: var(--sb-logo-size, 42px) !important;
    flex: 0 0 var(--sb-logo-size, 42px) !important;
}

.sb-brand__text {
    display: block !important;
    min-width: 0 !important;
    max-width: min(760px, calc(100vw - 160px)) !important;
    white-space: nowrap !important;
    overflow: hidden !important;
    text-overflow: ellipsis !important;
}

/* Перебиваем старое правило, которое уводило меню вправо */
.sb-public-header .sb-public-menu {
    margin-left: 0 !important;
}

@media (max-width: 760px) {
    .sb-header-brand-row {
        align-items: flex-start;
        flex-direction: column;
        gap: 10px;
    }

    .sb-brand {
        width: 100% !important;
    }

    .sb-brand__text {
        max-width: calc(100vw - 96px) !important;
    }

    .sb-header-menu-row {
        width: 100%;
    }

    .sb-header-menu-row .sb-public-menu {
        width: 100%;
    }
}


---

3. Обнови версию CSS

В public_page.php найди:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=3">

или похожую строку и поставь новую версию:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=4">

После этого обнови публичную страницу через Ctrl + F5.

Должно стать так: логотип и название слева в одной строке, длинное название обрезается многоточием, а страницы Домашняя / Диск идут ниже под логотипом и названием, а не справа.