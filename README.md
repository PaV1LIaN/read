Похоже, прошлый вариант слишком сложный и событие всё равно не цепляется. Давай сделаем проще и надёжнее: будем тянуть прямо за правый край заголовка столбца или за уже существующую ручку data-column-resizer.

1. Полностью замени table-edit.js

Файл:

/local/sitebuilder/assets/public/table-edit.js

Вставь весь файл целиком:

(function () {
    window.SB_TABLE_EDIT_LOADED = 'v4-edge-resize';

    var config = window.SB_PUBLIC_EDIT_CONFIG || {};
    var API_URL = config.apiUrl || '/local/sitebuilder/api.php';
    var sessid = config.sessid || '';

    var activeResize = null;

    function parseJson(value, fallback) {
        try {
            return JSON.parse(value || '');
        } catch (e) {
            return fallback;
        }
    }

    function cssEscape(value) {
        if (window.CSS && typeof window.CSS.escape === 'function') {
            return window.CSS.escape(value);
        }

        return String(value).replace(/"/g, '\\"');
    }

    function getClientX(e) {
        if (e.touches && e.touches[0]) {
            return e.touches[0].clientX;
        }

        if (e.changedTouches && e.changedTouches[0]) {
            return e.changedTouches[0].clientX;
        }

        return e.clientX;
    }

    function getContent(root) {
        return parseJson(root.getAttribute('data-content'), {});
    }

    function setContent(root, content) {
        root.setAttribute('data-content', JSON.stringify(content || {}));
    }

    function setDirty(root, isDirty) {
        root.classList.toggle('is-dirty', !!isDirty);

        var btn = root.querySelector('[data-table-save-widths]');

        if (btn) {
            btn.textContent = isDirty ? 'Сохранить ширину *' : 'Сохранить ширину';
        }
    }

    function clampWidth(width) {
        width = Math.round(Number(width || 0));

        if (width < 80) {
            width = 80;
        }

        if (width > 1200) {
            width = 1200;
        }

        return width;
    }

    function getColumnCurrentWidth(table, columnId) {
        var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

        if (!th) {
            return 160;
        }

        return clampWidth(th.getBoundingClientRect().width || 160);
    }

    function applyWidths(root) {
        var table = root.querySelector('.sb-public-table');
        var content = getContent(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];

        if (!table || !columns.length) {
            return;
        }

        var total = 0;

        columns.forEach(function (column) {
            var columnId = String(column.id || '');
            var width = clampWidth(column.width || getColumnCurrentWidth(table, columnId));

            column.width = width;
            total += width;

            var col = table.querySelector('col[data-column-id="' + cssEscape(columnId) + '"]');
            var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

            if (col) {
                col.style.setProperty('width', width + 'px', 'important');
                col.setAttribute('width', String(width));
            }

            if (th) {
                th.style.setProperty('width', width + 'px', 'important');
                th.style.setProperty('min-width', width + 'px', 'important');
                th.style.setProperty('max-width', width + 'px', 'important');
            }
        });

        table.style.setProperty('table-layout', 'fixed', 'important');
        table.style.setProperty('width', total + 'px', 'important');
        table.style.setProperty('min-width', total + 'px', 'important');

        content.columns = columns;
        setContent(root, content);
    }

    function updateColumnWidth(root, columnId, width) {
        var content = getContent(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];

        width = clampWidth(width);

        columns = columns.map(function (column) {
            if (String(column.id || '') === String(columnId)) {
                column.width = width;
            }

            return column;
        });

        content.columns = columns;
        setContent(root, content);

        applyWidths(root);
        setDirty(root, true);
    }

    function saveBlock(root) {
        var blockId = Number(root.getAttribute('data-block-id') || 0);
        var content = getContent(root);
        var props = parseJson(root.getAttribute('data-props'), {});

        if (!blockId) {
            alert('Не найден ID блока таблицы');
            return;
        }

        var formData = new FormData();

        formData.append('action', 'block.update');
        formData.append('sessid', sessid);
        formData.append('id', String(blockId));
        formData.append('content', JSON.stringify(content));
        formData.append('props', JSON.stringify(props || {}));

        var btn = root.querySelector('[data-table-save-widths]');

        if (btn) {
            btn.disabled = true;
            btn.textContent = 'Сохраняю...';
        }

        fetch(API_URL, {
            method: 'POST',
            body: formData,
            credentials: 'same-origin'
        })
            .then(function (response) {
                return response.json();
            })
            .then(function (res) {
                if (!res || !res.ok) {
                    throw new Error((res && (res.message || res.error)) || 'SAVE_ERROR');
                }

                setDirty(root, false);

                if (btn) {
                    btn.textContent = 'Сохранено';

                    setTimeout(function () {
                        btn.textContent = 'Сохранить ширину';
                    }, 1000);
                }
            })
            .catch(function (err) {
                console.error(err);
                alert('Не удалось сохранить ширину столбцов: ' + err.message);
                setDirty(root, true);
            })
            .finally(function () {
                if (btn) {
                    btn.disabled = false;
                }
            });
    }

    function startResize(e, th) {
        var root = th.closest('[data-public-editable-table]');
        var table = root ? root.querySelector('.sb-public-table') : null;
        var columnId = String(th.getAttribute('data-column-id') || '');

        if (!root || !table || !columnId) {
            return;
        }

        e.preventDefault();
        e.stopPropagation();

        activeResize = {
            root: root,
            table: table,
            columnId: columnId,
            startX: getClientX(e),
            startWidth: getColumnCurrentWidth(table, columnId)
        };

        document.body.classList.add('sb-public-table-resizing');
    }

    function moveResize(e) {
        if (!activeResize) {
            return;
        }

        e.preventDefault();

        var diff = getClientX(e) - activeResize.startX;
        var newWidth = activeResize.startWidth + diff;

        updateColumnWidth(activeResize.root, activeResize.columnId, newWidth);
    }

    function stopResize() {
        if (!activeResize) {
            return;
        }

        activeResize = null;
        document.body.classList.remove('sb-public-table-resizing');
    }

    function initTable(root) {
        applyWidths(root);

        root.querySelectorAll('[data-table-save-widths]').forEach(function (btn) {
            btn.addEventListener('click', function () {
                saveBlock(root);
            });
        });
    }

    function initAllTables() {
        document.querySelectorAll('[data-public-editable-table]').forEach(initTable);
    }

    document.addEventListener('mousedown', function (e) {
        var resizer = e.target.closest('[data-column-resizer]');

        if (resizer) {
            var thFromResizer = resizer.closest('th[data-column-id]');

            if (thFromResizer) {
                startResize(e, thFromResizer);
            }

            return;
        }

        var th = e.target.closest('.sb-public-table--editable th[data-column-id]');

        if (!th) {
            return;
        }

        var rect = th.getBoundingClientRect();
        var distanceFromRight = rect.right - e.clientX;

        if (distanceFromRight >= 0 && distanceFromRight <= 18) {
            startResize(e, th);
        }
    }, true);

    document.addEventListener('mousemove', moveResize, true);
    document.addEventListener('mouseup', stopResize, true);

    document.addEventListener('touchstart', function (e) {
        var resizer = e.target.closest('[data-column-resizer]');

        if (!resizer) {
            return;
        }

        var th = resizer.closest('th[data-column-id]');

        if (th) {
            startResize(e, th);
        }
    }, {capture: true, passive: false});

    document.addEventListener('touchmove', moveResize, {capture: true, passive: false});
    document.addEventListener('touchend', stopResize, true);
    document.addEventListener('touchcancel', stopResize, true);

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initAllTables);
    } else {
        initAllTables();
    }
})();

2. В конец public.css добавь

.sb-public-table--editable {
    table-layout: fixed !important;
    width: auto !important;
}

.sb-public-table--editable th[data-column-id] {
    position: relative !important;
    user-select: none !important;
}

.sb-public-table--editable th[data-column-id]:hover {
    cursor: default;
}

.sb-public-table-resizer {
    position: absolute !important;
    top: 0 !important;
    right: -8px !important;
    width: 18px !important;
    height: 100% !important;
    cursor: col-resize !important;
    z-index: 999 !important;
    touch-action: none !important;
    background: rgba(37, 99, 235, .08) !important;
}

.sb-public-table-resizer::after {
    content: "" !important;
    position: absolute !important;
    top: 8px !important;
    bottom: 8px !important;
    left: 8px !important;
    width: 2px !important;
    border-radius: 99px !important;
    background: rgba(37, 99, 235, .8) !important;
}

body.sb-public-table-resizing,
body.sb-public-table-resizing * {
    cursor: col-resize !important;
    user-select: none !important;
}

3. Обнови версии

В public_page.php поставь:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js?v=4"></script>

И CSS:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=16">

После Ctrl + F5 открой консоль и проверь:

window.SB_TABLE_EDIT_LOADED

Должно вернуть:

"v4-edge-resize"

Если вернёт undefined или старую версию — браузер не подключил новый файл, поэтому оно и не двигается.