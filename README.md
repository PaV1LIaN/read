Да, добавим удаление столбца. Раз у тебя уже работает table-edit.js, правим именно его и table.php.

1. В table.php добавь кнопку удаления столбца

Файл:

/local/sitebuilder/views/blocks/table.php

Найди внутри <th> вот этот кусок:

<select class="sb-public-table-align-select" data-column-align>
    <option value="left"<?= $column['align'] === 'left' ? ' selected' : '' ?>>Слева</option>
    <option value="center"<?= $column['align'] === 'center' ? ' selected' : '' ?>>Центр</option>
    <option value="right"<?= $column['align'] === 'right' ? ' selected' : '' ?>>Справа</option>
</select>

Сразу после него вставь:

<button
    class="sb-public-table-column-delete"
    type="button"
    data-table-delete-column
    title="Удалить столбец"
>
    Удалить столбец
</button>

Должно получиться примерно так:

<div class="sb-public-table-th-inner">
    <span
        class="sb-public-table__th-text"
        <?php if ($isEditMode): ?>
            contenteditable="true"
            data-column-label
        <?php endif; ?>
    ><?= sb_public_h($column['label']) ?></span>

    <?php if ($isEditMode): ?>
        <select class="sb-public-table-align-select" data-column-align>
            <option value="left"<?= $column['align'] === 'left' ? ' selected' : '' ?>>Слева</option>
            <option value="center"<?= $column['align'] === 'center' ? ' selected' : '' ?>>Центр</option>
            <option value="right"<?= $column['align'] === 'right' ? ' selected' : '' ?>>Справа</option>
        </select>

        <button
            class="sb-public-table-column-delete"
            type="button"
            data-table-delete-column
            title="Удалить столбец"
        >
            Удалить столбец
        </button>
    <?php endif; ?>
</div>


---

2. В table-edit.js добавь функцию удаления столбца

Файл:

/local/sitebuilder/assets/public/table-edit.js

Найди функцию:

function addColumn(root) {

После всей функции addColumn(root) { ... } вставь новую функцию:

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

    if (typeof applyWidths === 'function') {
        applyWidths(root);
    }

    if (typeof applyAllAligns === 'function') {
        applyAllAligns(root);
    }

    setContent(root, collectContentFromDom(root));
    setDirty(root, true);
}


---

3. В table-edit.js добавь обработчик кнопки

В этом же файле найди общий обработчик:

document.addEventListener('click', function (e) {

Внутри него, до обработки data-table-add-column, вставь:

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

То есть начало click-обработчика должно стать примерно таким:

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

    // дальше твой старый код: add column, add row, save, delete row...
}, true);

Если у тебя сначала идёт var root = ..., можно вставить удаление сразу после него, главное — до остальных обработок.


---

4. Если в addColumn() создаётся новый <th>, добавь кнопку туда тоже

В addColumn(root) найди кусок, где создаётся:

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

Замени на:

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


---

5. Добавь CSS

В конец файла:

/local/sitebuilder/assets/public/public.css

добавь:

.sb-public-table-column-delete {
    min-height: 26px;
    padding: 4px 8px;
    border: 0;
    border-radius: 8px;
    background: #fee2e2;
    color: #991b1b;
    font-size: 11px;
    font-weight: 900;
    cursor: pointer;
}

.sb-public-table-column-delete:hover {
    background: #fecaca;
}

После этого открой public с edit=Y, сделай Ctrl + F5, удали столбец и нажми “Сохранить изменения”.