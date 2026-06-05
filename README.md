Да, вот готовый table-edit.js целиком. Замени им файл:

/local/sitebuilder/assets/public/table-edit.js

(function () {
    window.SB_TABLE_EDIT_LOADED = 'v8-delete-column';

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

    function normalizeAlign(align) {
        align = String(align || 'left');

        if (align !== 'left' && align !== 'center' && align !== 'right') {
            return 'left';
        }

        return align;
    }

    function textValue(node) {
        return String(node ? (node.innerText || node.textContent || '') : '')
            .replace(/\u00a0/g, ' ')
            .trim();
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

        var btn = root.querySelector('[data-table-save-all]');

        if (btn) {
            btn.textContent = isDirty ? 'Сохранить изменения *' : 'Сохранить изменения';
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

    function getColumnAlignFromTh(th) {
        var select = th.querySelector('[data-column-align]');

        if (select) {
            return normalizeAlign(select.value);
        }

        return normalizeAlign(th.getAttribute('data-column-align-value') || 'left');
    }

    function collectContentFromDom(root) {
        var oldContent = getContent(root);
        var table = root.querySelector('.sb-public-table');
        var titleInput = root.querySelector('[data-table-title-input]');

        var columns = [];
        var rows = [];

        if (!table) {
            return oldContent;
        }

        table.querySelectorAll('thead th[data-column-id]').forEach(function (th, index) {
            var columnId = String(th.getAttribute('data-column-id') || '').trim();

            if (!columnId) {
                columnId = 'col_' + (index + 1);
                th.setAttribute('data-column-id', columnId);
            }

            var labelNode = th.querySelector('[data-column-label]') || th.querySelector('.sb-public-table__th-text');
            var label = textValue(labelNode);

            if (!label) {
                label = 'Столбец ' + (index + 1);
            }

            var oldColumn = null;

            if (Array.isArray(oldContent.columns)) {
                oldColumn = oldContent.columns.find(function (column) {
                    return String(column.id || '') === columnId;
                }) || null;
            }

            var width = oldColumn && oldColumn.width
                ? Number(oldColumn.width)
                : getColumnCurrentWidth(table, columnId);

            columns.push({
                id: columnId,
                label: label,
                width: clampWidth(width),
                align: getColumnAlignFromTh(th)
            });
        });

        table.querySelectorAll('tbody tr[data-row-id]').forEach(function (tr, rowIndex) {
            var rowId = String(tr.getAttribute('data-row-id') || '').trim();

            if (!rowId) {
                rowId = 'row_' + (Date.now() + rowIndex);
                tr.setAttribute('data-row-id', rowId);
            }

            var cells = {};

            columns.forEach(function (column) {
                var td = tr.querySelector('td[data-column-id="' + cssEscape(column.id) + '"]');
                cells[column.id] = textValue(td);
            });

            rows.push({
                id: rowId,
                cells: cells
            });
        });

        return {
            title: titleInput ? String(titleInput.value || '').trim() || 'Таблица' : (oldContent.title || 'Таблица'),
            columns: columns,
            rows: rows
        };
    }

    function applyColumnAlign(root, columnId, align) {
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        align = normalizeAlign(align);

        var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

        if (th) {
            th.style.textAlign = align;
            th.setAttribute('data-column-align-value', align);

            var select = th.querySelector('[data-column-align]');

            if (select) {
                select.value = align;
            }
        }

        table.querySelectorAll('td[data-column-id="' + cssEscape(columnId) + '"]').forEach(function (td) {
            td.style.textAlign = align;
        });
    }

    function applyAllAligns(root) {
        var content = getContent(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];

        columns.forEach(function (column) {
            applyColumnAlign(root, String(column.id || ''), normalizeAlign(column.align || 'left'));
        });
    }

    function applyWidths(root) {
        var table = root.querySelector('.sb-public-table');
        var content = getContent(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];

        if (!table || !columns.length) {
            return;
        }

        var total = root.querySelector('.sb-public-table__control-col') ? 72 : 0;

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
        var content = collectContentFromDom(root);
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
        applyAllAligns(root);
        setDirty(root, true);
    }

    function renumberRows(root) {
        root.querySelectorAll('tbody tr[data-row-id]').forEach(function (tr, index) {
            var num = tr.querySelector('.sb-public-table-row-num');

            if (num) {
                num.textContent = String(index + 1);
            }
        });
    }

    function updateEmptyRowColspan(root) {
        var table = root.querySelector('.sb-public-table');
        if (!table) return;

        var emptyRow = table.querySelector('tbody tr[data-empty-row]');
        if (!emptyRow) return;

        var td = emptyRow.querySelector('td');
        if (!td) return;

        var columnsCount = table.querySelectorAll('thead th[data-column-id]').length;
        var hasControlCol = !!root.querySelector('.sb-public-table__control-col');

        td.setAttribute('colspan', String(columnsCount + (hasControlCol ? 1 : 0)));
    }

    function ensureColumnControls(root) {
        root.querySelectorAll('thead th[data-column-id]').forEach(function (th) {
            var inner = th.querySelector('.sb-public-table-th-inner');

            if (!inner) {
                inner = document.createElement('div');
                inner.className = 'sb-public-table-th-inner';

                var textNode = th.querySelector('[data-column-label]') || th.querySelector('.sb-public-table__th-text');

                if (textNode) {
                    inner.appendChild(textNode);
                } else {
                    var span = document.createElement('span');
                    span.className = 'sb-public-table__th-text';
                    span.setAttribute('contenteditable', 'true');
                    span.setAttribute('data-column-label', '');
                    span.textContent = textValue(th) || 'Столбец';
                    inner.appendChild(span);
                }

                th.insertBefore(inner, th.firstChild);
            }

            if (!inner.querySelector('[data-column-align]')) {
                var align = normalizeAlign(th.getAttribute('data-column-align-value') || 'left');

                var select = document.createElement('select');
                select.className = 'sb-public-table-align-select';
                select.setAttribute('data-column-align', '');

                select.innerHTML = ''
                    + '<option value="left">Слева</option>'
                    + '<option value="center">Центр</option>'
                    + '<option value="right">Справа</option>';

                select.value = align;
                inner.appendChild(select);
            }

            if (!inner.querySelector('[data-table-delete-column]')) {
                var deleteBtn = document.createElement('button');
                deleteBtn.className = 'sb-public-table-column-delete';
                deleteBtn.type = 'button';
                deleteBtn.setAttribute('data-table-delete-column', '');
                deleteBtn.title = 'Удалить столбец';
                deleteBtn.textContent = 'Удалить столбец';

                inner.appendChild(deleteBtn);
            }
        });
    }

    function addColumn(root) {
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        var content = collectContentFromDom(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];
        var rows = Array.isArray(content.rows) ? content.rows : [];

        var newIndex = columns.length + 1;
        var newColumnId = 'col_' + Date.now();

        columns.push({
            id: newColumnId,
            label: 'Столбец ' + newIndex,
            width: 160,
            align: 'left'
        });

        rows = rows.map(function (row) {
            row.cells = row.cells || {};
            row.cells[newColumnId] = '';
            return row;
        });

        content.columns = columns;
        content.rows = rows;
        setContent(root, content);

        var colgroup = table.querySelector('colgroup');

        if (colgroup) {
            var col = document.createElement('col');
            col.setAttribute('data-column-id', newColumnId);
            col.setAttribute('width', '160');
            col.style.width = '160px';
            colgroup.appendChild(col);
        }

        var headRow = table.querySelector('thead tr');

        if (headRow) {
            var th = document.createElement('th');

            th.setAttribute('data-column-id', newColumnId);
            th.setAttribute('data-column-align-value', 'left');
            th.style.textAlign = 'left';

            th.innerHTML = ''
                + '<div class="sb-public-table-th-inner">'
                + '  <span class="sb-public-table__th-text" contenteditable="true" data-column-label>Столбец ' + newIndex + '</span>'
                + '  <select class="sb-public-table-align-select" data-column-align>'
                + '      <option value="left" selected>Слева</option>'
                + '      <option value="center">Центр</option>'
                + '      <option value="right">Справа</option>'
                + '  </select>'
                + '  <button class="sb-public-table-column-delete" type="button" data-table-delete-column title="Удалить столбец">Удалить столбец</button>'
                + '</div>'
                + '<span class="sb-public-table-resizer" data-column-resizer></span>';

            headRow.appendChild(th);
        }

        var tbody = table.querySelector('tbody');

        if (tbody) {
            var emptyRow = tbody.querySelector('[data-empty-row]');

            if (emptyRow) {
                emptyRow.remove();
            }

            tbody.querySelectorAll('tr[data-row-id]').forEach(function (tr) {
                var td = document.createElement('td');

                td.setAttribute('data-column-id', newColumnId);
                td.setAttribute('contenteditable', 'true');
                td.setAttribute('data-cell-editable', '');
                td.style.textAlign = 'left';

                tr.appendChild(td);
            });
        }

        applyWidths(root);
        applyAllAligns(root);
        updateEmptyRowColspan(root);

        setContent(root, collectContentFromDom(root));
        setDirty(root, true);
    }

    function deleteColumn(root, columnId) {
        var table = root.querySelector('.sb-public-table');

        if (!table || !columnId) {
            return;
        }

        var content = collectContentFromDom(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];
        var rows = Array.isArray(content.rows) ? content.rows : [];

        if (columns.length <= 1) {
            alert('Нельзя удалить последний столбец');
            return;
        }

        var column = columns.find(function (item) {
            return String(item.id || '') === String(columnId);
        });

        var columnName = column && column.label ? column.label : columnId;

        if (!confirm('Удалить столбец "' + columnName + '"? Данные в этом столбце будут удалены.')) {
            return;
        }

        columns = columns.filter(function (item) {
            return String(item.id || '') !== String(columnId);
        });

        rows = rows.map(function (row) {
            row.cells = row.cells || {};
            delete row.cells[columnId];
            return row;
        });

        content.columns = columns;
        content.rows = rows;

        setContent(root, content);

        var col = table.querySelector('col[data-column-id="' + cssEscape(columnId) + '"]');
        if (col) {
            col.remove();
        }

        var th = table.querySelector('thead th[data-column-id="' + cssEscape(columnId) + '"]');
        if (th) {
            th.remove();
        }

        table.querySelectorAll('tbody td[data-column-id="' + cssEscape(columnId) + '"]').forEach(function (td) {
            td.remove();
        });

        applyWidths(root);
        applyAllAligns(root);
        updateEmptyRowColspan(root);

        setContent(root, collectContentFromDom(root));
        setDirty(root, true);
    }

    function addRow(root) {
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        var content = collectContentFromDom(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];
        var tbody = table.querySelector('tbody');

        if (!tbody || !columns.length) {
            return;
        }

        var emptyRow = tbody.querySelector('[data-empty-row]');

        if (emptyRow) {
            emptyRow.remove();
        }

        var rowId = 'row_' + Date.now();
        var tr = document.createElement('tr');

        tr.setAttribute('data-row-id', rowId);

        var hasControlCol = !!root.querySelector('.sb-public-table__control-col');

        if (hasControlCol) {
            var controlTd = document.createElement('td');
            controlTd.className = 'sb-public-table__control-td';
            controlTd.innerHTML = ''
                + '<div class="sb-public-table-row-actions">'
                + '  <span class="sb-public-table-row-num"></span>'
                + '  <button type="button" class="sb-public-table-row-delete" data-table-delete-row title="Удалить строку">×</button>'
                + '</div>';

            tr.appendChild(controlTd);
        }

        columns.forEach(function (column) {
            var td = document.createElement('td');

            td.setAttribute('data-column-id', column.id);
            td.setAttribute('contenteditable', 'true');
            td.setAttribute('data-cell-editable', '');
            td.style.textAlign = normalizeAlign(column.align || 'left');

            tr.appendChild(td);
        });

        tbody.appendChild(tr);
        renumberRows(root);

        setContent(root, collectContentFromDom(root));
        setDirty(root, true);
    }

    function deleteRow(root, btn) {
        var tr = btn.closest('tr[data-row-id]');

        if (!tr) {
            return;
        }

        if (!confirm('Удалить строку?')) {
            return;
        }

        tr.remove();
        renumberRows(root);

        setContent(root, collectContentFromDom(root));
        setDirty(root, true);
    }

    function saveBlock(root) {
        var blockId = Number(root.getAttribute('data-block-id') || 0);
        var content = collectContentFromDom(root);
        var props = parseJson(root.getAttribute('data-props'), {});

        if (!blockId) {
            alert('Не найден ID блока таблицы');
            return;
        }

        setContent(root, content);

        var formData = new FormData();

        formData.append('action', 'block.update');
        formData.append('sessid', sessid);
        formData.append('id', String(blockId));
        formData.append('content', JSON.stringify(content));
        formData.append('props', JSON.stringify(props || {}));

        var btn = root.querySelector('[data-table-save-all]');

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
                        btn.textContent = 'Сохранить изменения';
                    }, 1000);
                }
            })
            .catch(function (err) {
                console.error(err);
                alert('Не удалось сохранить таблицу: ' + err.message);
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
        ensureColumnControls(root);
        applyWidths(root);
        applyAllAligns(root);
        renumberRows(root);
        updateEmptyRowColspan(root);
    }

    function initAllTables() {
        document.querySelectorAll('[data-public-editable-table]').forEach(initTable);
    }

    document.addEventListener('input', function (e) {
        var root = e.target.closest('[data-public-editable-table]');

        if (!root) {
            return;
        }

        if (
            e.target.matches('[data-table-title-input]') ||
            e.target.matches('[data-column-label]') ||
            e.target.matches('[data-cell-editable]')
        ) {
            setContent(root, collectContentFromDom(root));
            setDirty(root, true);
        }
    }, true);

    document.addEventListener('change', function (e) {
        var select = e.target.closest('[data-column-align]');

        if (!select) {
            return;
        }

        var root = select.closest('[data-public-editable-table]');
        var th = select.closest('th[data-column-id]');
        var columnId = th ? String(th.getAttribute('data-column-id') || '') : '';

        if (!root || !columnId) {
            return;
        }

        e.stopImmediatePropagation();

        applyColumnAlign(root, columnId, normalizeAlign(select.value));

        setContent(root, collectContentFromDom(root));
        setDirty(root, true);
    }, true);

    document.addEventListener('click', function (e) {
        var deleteColumnBtn = e.target.closest('[data-table-delete-column]');

        if (deleteColumnBtn) {
            var deleteRoot = deleteColumnBtn.closest('[data-public-editable-table]');
            var deleteTh = deleteColumnBtn.closest('th[data-column-id]');
            var deleteColumnId = deleteTh ? String(deleteTh.getAttribute('data-column-id') || '') : '';

            if (deleteRoot && deleteColumnId) {
                e.preventDefault();
                e.stopImmediatePropagation();
                deleteColumn(deleteRoot, deleteColumnId);
            }

            return;
        }

        var root = e.target.closest('[data-public-editable-table]');

        if (!root) {
            return;
        }

        if (e.target.closest('[data-table-add-column]')) {
            e.preventDefault();
            e.stopImmediatePropagation();
            addColumn(root);
            return;
        }

        if (e.target.closest('[data-table-add-row]')) {
            e.preventDefault();
            e.stopImmediatePropagation();
            addRow(root);
            return;
        }

        if (e.target.closest('[data-table-save-all]')) {
            e.preventDefault();
            e.stopImmediatePropagation();
            saveBlock(root);
            return;
        }

        var deleteBtn = e.target.closest('[data-table-delete-row]');

        if (deleteBtn) {
            e.preventDefault();
            e.stopImmediatePropagation();
            deleteRow(root, deleteBtn);
        }
    }, true);

    document.addEventListener('keydown', function (e) {
        if (e.target.matches('[data-cell-editable], [data-column-label]')) {
            if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                e.target.blur();
            }
        }
    }, true);

    document.addEventListener('mousedown', function (e) {
        if (e.target.closest('[data-column-align], [data-table-delete-column]')) {
            return;
        }

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

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initAllTables);
    } else {
        initAllTables();
    }
})();

Проверь после замены в консоли:

window.SB_TABLE_EDIT_LOADED

Должно вернуть:

"v8-delete-column"

Если у тебя кэшируется без версий, сделай именно Ctrl + F5 на public-странице с edit=Y.