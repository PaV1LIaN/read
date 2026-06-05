Понял. Раз ты не используешь версионность, лучше не мучить старый table-edit.js, потому что браузер может брать старый кэш. Сделаем новый файл-патч, который подключим после старого. Он исправит:

1. + Столбец
2. выравнивание слева / центр / справа
3. сохранение align в content.columns[].align

1. Создай новый файл

/local/sitebuilder/assets/public/table-edit-plus.js

Вставь туда:

(function () {
    window.SB_TABLE_EDIT_PLUS_LOADED = 'plus-1';

    var config = window.SB_PUBLIC_EDIT_CONFIG || {};
    var API_URL = config.apiUrl || '/local/sitebuilder/api.php';
    var sessid = config.sessid || '';

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

    function setDirty(root, isDirty) {
        root.classList.toggle('is-dirty', !!isDirty);

        var btn = root.querySelector('[data-table-save-all]');

        if (btn) {
            btn.textContent = isDirty ? 'Сохранить изменения *' : 'Сохранить изменения';
        }
    }

    function getOldColumn(root, columnId) {
        var content = parseJson(root.getAttribute('data-content'), {});
        var columns = Array.isArray(content.columns) ? content.columns : [];

        return columns.find(function (column) {
            return String(column.id || '') === String(columnId);
        }) || null;
    }

    function getColumnWidth(root, table, columnId) {
        var oldColumn = getOldColumn(root, columnId);

        if (oldColumn && oldColumn.width) {
            return Number(oldColumn.width) || 160;
        }

        var th = table.querySelector('th[data-column-id="' + cssEscape(columnId) + '"]');

        if (!th) {
            return 160;
        }

        var width = Math.round(th.getBoundingClientRect().width || 160);

        if (width < 80) {
            width = 80;
        }

        if (width > 1200) {
            width = 1200;
        }

        return width;
    }

    function collectTableContent(root) {
        var oldContent = parseJson(root.getAttribute('data-content'), {});
        var table = root.querySelector('.sb-public-table');
        var titleInput = root.querySelector('[data-table-title-input]');

        if (!table) {
            return oldContent;
        }

        var columns = [];
        var rows = [];

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

            var select = th.querySelector('[data-column-align]');
            var align = select
                ? normalizeAlign(select.value)
                : normalizeAlign(th.getAttribute('data-column-align-value') || 'left');

            columns.push({
                id: columnId,
                label: label,
                width: getColumnWidth(root, table, columnId),
                align: align
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

    function setContent(root, content) {
        root.setAttribute('data-content', JSON.stringify(content || {}));
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

        setContent(root, collectTableContent(root));
        setDirty(root, true);
    }

    function addColumn(root) {
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        var content = collectTableContent(root);
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
            th.style.width = '160px';
            th.style.minWidth = '160px';
            th.style.maxWidth = '160px';

            th.innerHTML = ''
                + '<div class="sb-public-table-th-inner">'
                + '  <span class="sb-public-table__th-text" contenteditable="true" data-column-label>Столбец ' + newIndex + '</span>'
                + '  <select class="sb-public-table-align-select" data-column-align>'
                + '      <option value="left" selected>Слева</option>'
                + '      <option value="center">Центр</option>'
                + '      <option value="right">Справа</option>'
                + '  </select>'
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

        setContent(root, collectTableContent(root));
        setDirty(root, true);
    }

    function saveTable(root) {
        var blockId = Number(root.getAttribute('data-block-id') || 0);
        var content = collectTableContent(root);
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

    document.addEventListener('click', function (e) {
        var addColumnBtn = e.target.closest('[data-table-add-column]');

        if (addColumnBtn) {
            var root = addColumnBtn.closest('[data-public-editable-table]');

            if (root) {
                e.preventDefault();
                e.stopImmediatePropagation();
                addColumn(root);
            }

            return;
        }

        var saveBtn = e.target.closest('[data-table-save-all]');

        if (saveBtn) {
            var saveRoot = saveBtn.closest('[data-public-editable-table]');

            if (saveRoot) {
                e.preventDefault();
                e.stopImmediatePropagation();
                saveTable(saveRoot);
            }

            return;
        }
    }, true);

    document.addEventListener('change', function (e) {
        var select = e.target.closest('[data-column-align]');

        if (!select) {
            return;
        }

        var root = select.closest('[data-public-editable-table]');
        var th = select.closest('th[data-column-id]');

        if (!root || !th) {
            return;
        }

        e.stopImmediatePropagation();

        var columnId = String(th.getAttribute('data-column-id') || '');
        var align = normalizeAlign(select.value);

        applyColumnAlign(root, columnId, align);
    }, true);

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
            setContent(root, collectTableContent(root));
            setDirty(root, true);
        }
    }, true);

    document.addEventListener('DOMContentLoaded', function () {
        document.querySelectorAll('[data-public-editable-table]').forEach(function (root) {
            var content = collectTableContent(root);

            setContent(root, content);

            content.columns.forEach(function (column) {
                applyColumnAlign(root, column.id, column.align || 'left');
            });

            setDirty(root, false);
        });
    });
})();

2. Подключи этот файл после table-edit.js

В файле:

/local/sitebuilder/views/layout/public_page.php

Там, где подключён:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js"></script>

сразу после него добавь:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit-plus.js"></script>

Раз ты версионность не используешь, новый файл должен подхватиться сразу.


---

3. Проверь в консоли

После Ctrl + F5 на public-странице с edit=Y выполни:

window.SB_TABLE_EDIT_PLUS_LOADED

Должно вернуть:

"plus-1"

Если вернёт undefined, значит table-edit-plus.js не подключился.


---

4. Почему выравнивание сохранялось как “слева”

Старый обработчик сохранения собирал таблицу, но не забирал значение из:

<select data-column-align>

Поэтому в content.columns[].align попадало значение по умолчанию:

"align": "left"

Новый table-edit-plus.js перехватывает сохранение и сохраняет align явно.