Да, сейчас они выглядят неудобно, потому что элементы идут почти столбиком:

Обновить / Настройки
Файлы
Поиск
Сортировка
Загрузить / Новая папка
Таблица / Плитка

Сделаем нормальную панель:

[Поиск файлов и папок........] [Сначала новые]
                                      [Загрузить] [Новая папка] [Обновить] [Настройки] [Таблица] [Плитка]

1. В конец /local/sitebuilder/components/disk/script.js

Добавь:

/* =========================================================
   SITEBUILDER DISK TOOLBAR NORMALIZER
   Делает кнопки и поля удобной панелью
   ========================================================= */

(function () {
    'use strict';

    function cleanText(node) {
        return String(node && node.textContent ? node.textContent : '').trim().toLowerCase();
    }

    function isInsideModernPanel(node) {
        return !!(node && node.closest && node.closest('.sb-disk-modern-panel'));
    }

    function isVisible(node) {
        if (!node) return false;

        var rect = node.getBoundingClientRect();
        return rect.width > 0 || rect.height > 0;
    }

    function getOrCreatePanel(disk) {
        var panel = disk.querySelector(':scope > .sb-disk-modern-panel');

        if (panel) {
            return panel;
        }

        panel = document.createElement('div');
        panel.className = 'sb-disk-modern-panel';
        panel.innerHTML = ''
            + '<div class="sb-disk-modern-row">'
            + '  <div class="sb-disk-modern-filters"></div>'
            + '  <div class="sb-disk-modern-actions"></div>'
            + '</div>';

        var firstTable = disk.querySelector('table, .sb-disk-table-wrap, .sb-disk-table-container');
        var firstEmpty = disk.querySelector('.sb-public-disk-loading, .sb-disk-empty, .sb-disk-empty-state');

        var beforeNode = firstTable || firstEmpty || disk.firstChild;

        if (beforeNode) {
            disk.insertBefore(panel, beforeNode);
        } else {
            disk.appendChild(panel);
        }

        return panel;
    }

    function moveNode(target, node) {
        if (!node || !target) return;
        if (isInsideModernPanel(node)) return;

        target.appendChild(node);
    }

    function normalizeDiskToolbar(disk) {
        if (!disk) return;

        var panel = getOrCreatePanel(disk);
        var filters = panel.querySelector('.sb-disk-modern-filters');
        var actions = panel.querySelector('.sb-disk-modern-actions');

        if (!filters || !actions) return;

        var inputs = Array.prototype.slice.call(disk.querySelectorAll('input[type="text"], input[type="search"]'));
        inputs.forEach(function (input) {
            if (isInsideModernPanel(input)) return;

            var placeholder = String(input.getAttribute('placeholder') || '').toLowerCase();

            if (placeholder.indexOf('поиск') !== -1 || placeholder === '') {
                input.classList.add('sb-disk-modern-search');

                if (!input.getAttribute('placeholder')) {
                    input.setAttribute('placeholder', 'Поиск файлов и папок');
                }

                moveNode(filters, input);
            }
        });

        var selects = Array.prototype.slice.call(disk.querySelectorAll('select'));
        selects.forEach(function (select) {
            if (isInsideModernPanel(select)) return;

            select.classList.add('sb-disk-modern-select');
            moveNode(filters, select);
        });

        var buttons = Array.prototype.slice.call(disk.querySelectorAll('button'));
        buttons.forEach(function (button) {
            if (isInsideModernPanel(button)) return;

            var label = cleanText(button);

            if (!label) return;

            if (label.indexOf('загруз') !== -1) {
                button.classList.add('sb-disk-modern-btn', 'sb-disk-modern-btn-primary');
                moveNode(actions, button);
                return;
            }

            if (label.indexOf('новая пап') !== -1 || label.indexOf('создать пап') !== -1) {
                button.classList.add('sb-disk-modern-btn');
                moveNode(actions, button);
                return;
            }

            if (label.indexOf('обнов') !== -1 || label.indexOf('настрой') !== -1) {
                button.classList.add('sb-disk-modern-btn');
                moveNode(actions, button);
                return;
            }

            if (label.indexOf('таблица') !== -1 || label.indexOf('плитка') !== -1) {
                button.classList.add('sb-disk-modern-btn', 'sb-disk-modern-btn-view');
                moveNode(actions, button);
            }
        });

        disk.classList.add('sb-disk-toolbar-normalized');
    }

    function normalizeAll() {
        document.querySelectorAll('.sb-disk').forEach(normalizeDiskToolbar);
    }

    function startObserver() {
        var timer = null;

        var observer = new MutationObserver(function () {
            clearTimeout(timer);

            timer = setTimeout(function () {
                normalizeAll();
            }, 120);
        });

        observer.observe(document.body, {
            childList: true,
            subtree: true
        });
    }

    document.addEventListener('DOMContentLoaded', function () {
        normalizeAll();
        startObserver();
    });

    if (document.readyState === 'interactive' || document.readyState === 'complete') {
        normalizeAll();
        startObserver();
    }
})();


---

2. В конец /local/sitebuilder/components/disk/styles.css

Добавь:

/* =========================================================
   DISK MODERN TOOLBAR FIX
   Поля и кнопки в удобной панели
   ========================================================= */

.sb-disk-modern-panel {
    width: 100%;
    margin: 16px 0 18px;
    padding: 14px;
    border: 1px solid #e5e7eb;
    border-radius: 18px;
    background: #f8fafc;
    box-shadow: inset 0 1px 0 rgba(255, 255, 255, .8);
}

.sb-disk-modern-row {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    width: 100%;
}

.sb-disk-modern-filters {
    display: flex;
    align-items: center;
    gap: 10px;
    flex: 1 1 auto;
    min-width: 0;
}

.sb-disk-modern-actions {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 8px;
    flex: 0 0 auto;
    flex-wrap: wrap;
}

.sb-disk-modern-panel .sb-disk-modern-search,
.sb-disk-modern-panel input[type="text"],
.sb-disk-modern-panel input[type="search"] {
    width: min(420px, 100%) !important;
    min-width: 260px !important;
    height: 40px !important;
    padding: 0 14px !important;
    border: 1px solid #dbe3ef !important;
    border-radius: 12px !important;
    background: #fff !important;
    color: #111827 !important;
    font-size: 13px !important;
    outline: none !important;
}

.sb-disk-modern-panel .sb-disk-modern-select,
.sb-disk-modern-panel select {
    width: 180px !important;
    min-width: 180px !important;
    height: 40px !important;
    padding: 0 12px !important;
    border: 1px solid #dbe3ef !important;
    border-radius: 12px !important;
    background: #fff !important;
    color: #111827 !important;
    font-size: 13px !important;
    outline: none !important;
}

.sb-disk-modern-panel input:focus,
.sb-disk-modern-panel select:focus {
    border-color: var(--disk-accent, #2563eb) !important;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12) !important;
}

.sb-disk-modern-panel .sb-disk-modern-btn,
.sb-disk-modern-panel button {
    width: auto !important;
    min-width: auto !important;
    height: 40px !important;
    min-height: 40px !important;
    padding: 0 14px !important;
    border: 1px solid #dbe3ef !important;
    border-radius: 12px !important;
    background: #fff !important;
    color: #374151 !important;
    font-size: 13px !important;
    font-weight: 700 !important;
    white-space: nowrap !important;
    box-shadow: none !important;
}

.sb-disk-modern-panel .sb-disk-modern-btn:hover,
.sb-disk-modern-panel button:hover {
    border-color: #c7d2fe !important;
    background: #eef2ff !important;
    color: var(--disk-accent, #2563eb) !important;
}

.sb-disk-modern-panel .sb-disk-modern-btn-primary,
.sb-disk-modern-panel .sb-disk-upload-btn,
.sb-disk-modern-panel .is-primary {
    border-color: var(--disk-accent, #2563eb) !important;
    background: var(--disk-accent, #2563eb) !important;
    color: #fff !important;
}

.sb-disk-modern-panel .sb-disk-modern-btn-primary:hover,
.sb-disk-modern-panel .sb-disk-upload-btn:hover,
.sb-disk-modern-panel .is-primary:hover {
    background: var(--disk-accent, #2563eb) !important;
    color: #fff !important;
    box-shadow: 0 10px 22px rgba(37, 99, 235, .22) !important;
}

.sb-disk-modern-panel .sb-disk-modern-btn-view {
    padding: 0 12px !important;
}

/* Убираем старые пустые отступы, если кнопки были перенесены */
.sb-disk-toolbar-normalized .sb-disk-toolbar:empty,
.sb-disk-toolbar-normalized .sb-disk-actions:empty,
.sb-disk-toolbar-normalized .sb-disk-controls:empty,
.sb-disk-toolbar-normalized .sb-disk-filter:empty,
.sb-disk-toolbar-normalized .sb-disk-filters:empty,
.sb-disk-toolbar-normalized .sb-disk-search-row:empty {
    display: none !important;
}

/* Старые кнопки не должны растягиваться на всю ширину */
.sb-disk-toolbar-normalized button {
    width: auto;
}

/* Адаптив */
@media (max-width: 1100px) {
    .sb-disk-modern-row {
        align-items: stretch;
        flex-direction: column;
    }

    .sb-disk-modern-filters {
        width: 100%;
    }

    .sb-disk-modern-actions {
        width: 100%;
        justify-content: flex-start;
    }

    .sb-disk-modern-panel .sb-disk-modern-search,
    .sb-disk-modern-panel input[type="text"],
    .sb-disk-modern-panel input[type="search"] {
        flex: 1 1 auto;
        min-width: 220px !important;
    }
}

@media (max-width: 700px) {
    .sb-disk-modern-panel {
        padding: 12px;
    }

    .sb-disk-modern-filters,
    .sb-disk-modern-actions {
        flex-direction: column;
        align-items: stretch;
    }

    .sb-disk-modern-panel .sb-disk-modern-search,
    .sb-disk-modern-panel input[type="text"],
    .sb-disk-modern-panel input[type="search"],
    .sb-disk-modern-panel .sb-disk-modern-select,
    .sb-disk-modern-panel select,
    .sb-disk-modern-panel button {
        width: 100% !important;
        min-width: 0 !important;
    }
}


---

3. Обнови версии подключения

В public_page.php:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=4">

и:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=3"></script>

После этого сделай Ctrl + F5.

Должно стать удобнее: поиск и сортировка в одной строке, кнопки рядом справа, без этого “лесенкой вниз”.