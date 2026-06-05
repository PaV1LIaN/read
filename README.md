Понял. Скорее всего ширина ставится на th/col, но таблица остаётся width:100%, поэтому браузер обратно растягивает колонки. Давай заменим table-edit.js на более жёсткий вариант: он будет задавать ширину каждому col и самой таблице в пикселях.

1. Полностью замени table-edit.js

Файл:

/local/sitebuilder/assets/public/table-edit.js

Замени весь файл на это:

(function () {
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

    function setDirty(root, isDirty) {
        root.classList.toggle('is-dirty', !!isDirty);

        var btn = root.querySelector('[data-table-save-widths]');

        if (btn) {
            btn.textContent = isDirty ? 'Сохранить ширину *' : 'Сохранить ширину';
        }
    }

    function getTableColumns(root) {
        var content = parseJson(root.getAttribute('data-content'), {});
        return Array.isArray(content.columns) ? content.columns : [];
    }

    function setTableColumns(root, columns) {
        var content = parseJson(root.getAttribute('data-content'), {});

        content.columns = columns;

        root.setAttribute('data-content', JSON.stringify(content));
    }

    function applyTablePixelWidth(table) {
        var cols = Array.prototype.slice.call(table.querySelectorAll('col[data-column-id]'));

        if (!cols.length) {
            return;
        }

        var total = 0;

        cols.forEach(function (col) {
            var width = parseInt(col.style.width || col.getAttribute('width') || '0', 10);

            if (!width || width < 80) {
                var columnId = col.getAttribute('data-column-id');
                var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

                width = th ? Math.round(th.getBoundingClientRect().width) : 160;
            }

            if (width < 80) {
                width = 80;
            }

            col.style.width = width + 'px';
            total += width;
        });

        if (total > 0) {
            table.style.width = total + 'px';
            table.style.minWidth = total + 'px';
            table.style.tableLayout = 'fixed';
        }
    }

    function initColumnWidths(root) {
        var table = root.querySelector('.sb-public-table');
        var columns = getTableColumns(root);

        if (!table || !columns.length) {
            return;
        }

        columns = columns.map(function (column) {
            var columnId = String(column.id || '');
            var col = table.querySelector('col[data-column-id="' + cssEscape(columnId) + '"]');
            var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

            var width = parseInt(column.width || 0, 10);

            if (!width || width < 80) {
                width = th ? Math.round(th.getBoundingClientRect().width) : 160;
            }

            if (width < 80) {
                width = 80;
            }

            if (width > 1200) {
                width = 1200;
            }

            column.width = width;

            if (col) {
                col.style.width = width + 'px';
            }

            if (th) {
                th.style.width = width + 'px';
                th.style.minWidth = width + 'px';
                th.style.maxWidth = width + 'px';
            }

            return column;
        });

        setTableColumns(root, columns);
        applyTablePixelWidth(table);
    }

    function updateColumnWidth(root, columnId, newWidth) {
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        if (newWidth < 80) {
            newWidth = 80;
        }

        if (newWidth > 1200) {
            newWidth = 1200;
        }

        var col = table.querySelector('col[data-column-id="' + cssEscape(columnId) + '"]');
        var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

        if (col) {
            col.style.width = newWidth + 'px';
        }

        if (th) {
            th.style.width = newWidth + 'px';
            th.style.minWidth = newWidth + 'px';
            th.style.maxWidth = newWidth + 'px';
        }

        var columns = getTableColumns(root).map(function (column) {
            if (String(column.id || '') === String(columnId)) {
                column.width = newWidth;
            }

            return column;
        });

        setTableColumns(root, columns);
        applyTablePixelWidth(table);
        setDirty(root, true);
    }

    function saveBlock(root) {
        var blockId = Number(root.getAttribute('data-block-id') || 0);
        var content = parseJson(root.getAttribute('data-content'), {});
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

    function getClientX(e) {
        if (e.touches && e.touches[0]) {
            return e.touches[0].clientX;
        }

        if (e.changedTouches && e.changedTouches[0]) {
            return e.changedTouches[0].clientX;
        }

        return e.clientX;
    }

    function startResize(e, resizer) {
        e.preventDefault();
        e.stopPropagation();

        var root = resizer.closest('[data-public-editable-table]');
        var th = resizer.closest('th[data-column-id]');

        if (!root || !th) {
            return;
        }

        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        var columnId = String(th.getAttribute('data-column-id') || '');
        var startWidth = Math.round(th.getBoundingClientRect().width);

        if (!startWidth || startWidth < 80) {
            startWidth = 160;
        }

        activeResize = {
            root: root,
            table: table,
            th: th,
            columnId: columnId,
            startX: getClientX(e),
            startWidth: startWidth
        };

        document.body.classList.add('sb-public-table-resizing');

        if (resizer.setPointerCapture && e.pointerId) {
            try {
                resizer.setPointerCapture(e.pointerId);
            } catch (err) {}
        }
    }

    function moveResize(e) {
        if (!activeResize) {
            return;
        }

        e.preventDefault();

        var diff = getClientX(e) - activeResize.startX;
        var newWidth = Math.round(activeResize.startWidth + diff);

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
        initColumnWidths(root);

        var saveBtn = root.querySelector('[data-table-save-widths]');

        if (saveBtn) {
            saveBtn.addEventListener('click', function () {
                saveBlock(root);
            });
        }
    }

    function initAllTables() {
        document.querySelectorAll('[data-public-editable-table]').forEach(initTable);
    }

    document.addEventListener('pointerdown', function (e) {
        var resizer = e.target.closest('[data-column-resizer]');

        if (!resizer) {
            return;
        }

        startResize(e, resizer);
    });

    document.addEventListener('pointermove', moveResize);
    document.addEventListener('pointerup', stopResize);
    document.addEventListener('pointercancel', stopResize);

    document.addEventListener('mousedown', function (e) {
        var resizer = e.target.closest('[data-column-resizer]');

        if (!resizer) {
            return;
        }

        startResize(e, resizer);
    });

    document.addEventListener('mousemove', moveResize);
    document.addEventListener('mouseup', stopResize);

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initAllTables);
    } else {
        initAllTables();
    }
})();


---

2. Добавь/замени CSS для ручки

В файл:

/local/sitebuilder/assets/public/public.css

в самый конец добавь:

.sb-public-table--editable {
    table-layout: fixed !important;
    width: auto;
}

.sb-public-table--editable th {
    position: relative;
    user-select: none;
    overflow: visible;
}

.sb-public-table-resizer {
    position: absolute;
    top: 0;
    right: -6px;
    width: 14px;
    height: 100%;
    cursor: col-resize;
    z-index: 50;
    touch-action: none;
}

.sb-public-table-resizer::after {
    content: "";
    position: absolute;
    top: 8px;
    bottom: 8px;
    left: 6px;
    width: 2px;
    border-radius: 99px;
    background: rgba(37, 99, 235, .55);
    opacity: .35;
}

.sb-public-table-resizer:hover::after,
body.sb-public-table-resizing .sb-public-table-resizer::after {
    opacity: 1;
    background: rgba(37, 99, 235, .95);
}

body.sb-public-table-resizing,
body.sb-public-table-resizing * {
    cursor: col-resize !important;
    user-select: none !important;
}


---

3. Обнови версии в public_page.php

Поменяй подключение JS:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js?v=2"></script>

И CSS:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=14">

Потом открой public с edit=Y и сделай Ctrl + F5.


---

4. Как тянуть

Тянуть нужно не за середину заголовка, а за правую границу заголовка столбца. После замены там должна появляться синяя вертикальная полоска при наведении.