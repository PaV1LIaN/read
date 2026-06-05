Похоже, ручка внутри th не ловит событие. Давай сделаем над таблицей отдельные широкие невидимые “ручки”, которые точно будут ловить мышку.

1. Полностью замени table-edit.js

Файл:

/local/sitebuilder/assets/public/table-edit.js

Замени весь код на этот:

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

    function getColumnWidth(table, columnId) {
        var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

        if (!th) {
            return 160;
        }

        var width = Math.round(th.getBoundingClientRect().width);

        if (!width || width < 80) {
            width = 160;
        }

        return width;
    }

    function applyTableWidth(root) {
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        var content = getContent(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];
        var total = 0;

        columns.forEach(function (column) {
            var columnId = String(column.id || '');
            var width = parseInt(column.width || 0, 10);

            if (!width || width < 80) {
                width = getColumnWidth(table, columnId);
            }

            if (width < 80) {
                width = 80;
            }

            if (width > 1200) {
                width = 1200;
            }

            column.width = width;
            total += width;

            var col = table.querySelector('col[data-column-id="' + cssEscape(columnId) + '"]');
            var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

            if (col) {
                col.style.width = width + 'px';
            }

            if (th) {
                th.style.width = width + 'px';
                th.style.minWidth = width + 'px';
                th.style.maxWidth = width + 'px';
            }
        });

        if (total > 0) {
            table.style.tableLayout = 'fixed';
            table.style.width = total + 'px';
            table.style.minWidth = total + 'px';
        }

        content.columns = columns;
        setContent(root, content);
    }

    function updateColumnWidth(root, columnId, width) {
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        width = Math.round(width);

        if (width < 80) {
            width = 80;
        }

        if (width > 1200) {
            width = 1200;
        }

        var content = getContent(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];

        columns = columns.map(function (column) {
            if (String(column.id || '') === String(columnId)) {
                column.width = width;
            }

            return column;
        });

        content.columns = columns;
        setContent(root, content);

        applyTableWidth(root);
        buildResizeLayer(root);
        setDirty(root, true);
    }

    function removeResizeLayer(root) {
        root.querySelectorAll('.sb-public-table-resize-layer').forEach(function (node) {
            node.remove();
        });
    }

    function buildResizeLayer(root) {
        var wrap = root.querySelector('.sb-public-table-wrap');
        var table = root.querySelector('.sb-public-table');

        if (!wrap || !table) {
            return;
        }

        removeResizeLayer(root);

        wrap.style.position = 'relative';

        var layer = document.createElement('div');
        layer.className = 'sb-public-table-resize-layer';

        var wrapRect = wrap.getBoundingClientRect();
        var tableRect = table.getBoundingClientRect();

        layer.style.position = 'absolute';
        layer.style.left = '0';
        layer.style.top = '0';
        layer.style.width = Math.max(wrap.scrollWidth, wrap.clientWidth) + 'px';
        layer.style.height = Math.max(table.offsetHeight, tableRect.height) + 'px';
        layer.style.pointerEvents = 'none';
        layer.style.zIndex = '20';

        table.querySelectorAll('th[data-column-id]').forEach(function (th) {
            var columnId = String(th.getAttribute('data-column-id') || '');

            if (!columnId) {
                return;
            }

            var thRect = th.getBoundingClientRect();

            var handle = document.createElement('div');
            handle.className = 'sb-public-table-resize-handle';
            handle.setAttribute('data-resize-column-id', columnId);

            var left = (thRect.right - wrapRect.left) + wrap.scrollLeft - 8;
            var top = (tableRect.top - wrapRect.top) + wrap.scrollTop;

            handle.style.position = 'absolute';
            handle.style.left = left + 'px';
            handle.style.top = top + 'px';
            handle.style.width = '16px';
            handle.style.height = Math.max(table.offsetHeight, tableRect.height) + 'px';
            handle.style.cursor = 'col-resize';
            handle.style.pointerEvents = 'auto';
            handle.style.touchAction = 'none';

            layer.appendChild(handle);
        });

        wrap.appendChild(layer);
    }

    function startResize(e, handle) {
        e.preventDefault();
        e.stopPropagation();

        var root = handle.closest('[data-public-editable-table]');
        var columnId = String(handle.getAttribute('data-resize-column-id') || '');

        if (!root || !columnId) {
            return;
        }

        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        activeResize = {
            root: root,
            table: table,
            columnId: columnId,
            startX: getClientX(e),
            startWidth: getColumnWidth(table, columnId)
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

        buildResizeLayer(activeResize.root);

        activeResize = null;
        document.body.classList.remove('sb-public-table-resizing');
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

    function initTable(root) {
        applyTableWidth(root);
        buildResizeLayer(root);

        var wrap = root.querySelector('.sb-public-table-wrap');

        if (wrap) {
            wrap.addEventListener('scroll', function () {
                buildResizeLayer(root);
            });
        }

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

    document.addEventListener('mousedown', function (e) {
        var handle = e.target.closest('.sb-public-table-resize-handle');

        if (!handle) {
            return;
        }

        startResize(e, handle);
    }, true);

    document.addEventListener('mousemove', moveResize, true);
    document.addEventListener('mouseup', stopResize, true);

    document.addEventListener('touchstart', function (e) {
        var handle = e.target.closest('.sb-public-table-resize-handle');

        if (!handle) {
            return;
        }

        startResize(e, handle);
    }, {capture: true, passive: false});

    document.addEventListener('touchmove', moveResize, {capture: true, passive: false});
    document.addEventListener('touchend', stopResize, true);
    document.addEventListener('touchcancel', stopResize, true);

    window.addEventListener('resize', function () {
        document.querySelectorAll('[data-public-editable-table]').forEach(function (root) {
            buildResizeLayer(root);
        });
    });

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initAllTables);
    } else {
        initAllTables();
    }
})();

2. Добавь стили поверх старых

В конец:

/local/sitebuilder/assets/public/public.css

добавь:

.sb-public-table-resize-layer {
    pointer-events: none;
}

.sb-public-table-resize-handle {
    background: rgba(37, 99, 235, .06);
    border-left: 2px solid rgba(37, 99, 235, .45);
}

.sb-public-table-resize-handle:hover {
    background: rgba(37, 99, 235, .14);
    border-left-color: rgba(37, 99, 235, .95);
}

body.sb-public-table-resizing,
body.sb-public-table-resizing * {
    cursor: col-resize !important;
    user-select: none !important;
}

3. Обнови версию подключения

В public_page.php поставь:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js?v=3"></script>

И CSS:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=15">

Потом Ctrl + F5.

Теперь на таблице должны появиться синие вертикальные полоски по границам столбцов. Тянуть нужно именно за них.