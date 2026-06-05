Да, сейчас данные заполняются в узкой правой панели, поэтому неудобно. Сделаем большое окно редактирования таблицы, как мини-Excel:

Открыть таблицу → большое окно → столбцы сверху → строки ниже → заполнение ячеек

1. В 30-blocks.js замени весь блок TABLE BLOCK

Файл:

/local/sitebuilder/assets/admin/editor/30-blocks.js

Найди место:

/* =========================================================
   TABLE BLOCK
   ========================================================= */

И замени весь блок от этого комментария до function fillVisualBlockForm(block) на этот код:

/* =========================================================
   TABLE BLOCK
   ========================================================= */

var tableEditorDraft = null;

function normalizeTableContent(content) {
    content = content || {};

    var columns = Array.isArray(content.columns) ? content.columns : [];
    var rows = Array.isArray(content.rows) ? content.rows : [];

    if (!columns.length) {
        columns = [
            {id: 'col_1', label: 'Столбец 1'},
            {id: 'col_2', label: 'Столбец 2'},
            {id: 'col_3', label: 'Столбец 3'}
        ];
    }

    columns = columns.map(function (column, index) {
        var id = String(column.id || '').trim();

        if (!id) {
            id = 'col_' + (index + 1);
        }

        return {
            id: id,
            label: String(column.label || ('Столбец ' + (index + 1)))
        };
    });

    rows = rows.map(function (row) {
        var cells = row && row.cells && typeof row.cells === 'object' ? row.cells : {};

        return {
            id: String((row && row.id) || ('row_' + Date.now() + '_' + Math.random().toString(16).slice(2))),
            cells: cells
        };
    });

    return {
        title: String(content.title || 'Таблица'),
        columns: columns,
        rows: rows
    };
}

function renderTableEditor(content) {
    tableEditorDraft = normalizeTableContent(content);

    var titleInput = document.getElementById('tableTitleInput');
    var columnsNode = document.getElementById('tableColumnsEditor');
    var rowsNode = document.getElementById('tableRowsEditor');

    if (titleInput) {
        titleInput.value = tableEditorDraft.title || '';
    }

    if (columnsNode) {
        columnsNode.innerHTML = ''
            + '<div class="sb-table-editor__summary">'
            + '  <div><strong>Столбцов:</strong> ' + tableEditorDraft.columns.length + '</div>'
            + '  <div class="sb-table-editor__chips">'
            + tableEditorDraft.columns.map(function (column) {
                return '<span class="sb-table-editor__chip">' + escapeHtml(column.label) + '</span>';
            }).join('')
            + '  </div>'
            + '</div>';
    }

    if (rowsNode) {
        rowsNode.innerHTML = ''
            + '<div class="sb-table-editor__summary">'
            + '  <div><strong>Строк:</strong> ' + tableEditorDraft.rows.length + '</div>'
            + '  <button class="sb-btn sb-btn-primary" type="button" data-table-action="open-data-modal">'
            + '      Открыть удобное заполнение'
            + '  </button>'
            + '</div>';
    }
}

function collectTableContentFromEditor() {
    var titleInput = document.getElementById('tableTitleInput');

    var current = normalizeTableContent(tableEditorDraft || {});

    current.title = titleInput
        ? String(titleInput.value || '').trim()
        : current.title;

    if (!current.title) {
        current.title = 'Таблица';
    }

    return current;
}

function ensureTableDataModal() {
    var modal = document.getElementById('sbTableDataModal');

    if (modal) {
        return modal;
    }

    modal = document.createElement('div');
    modal.id = 'sbTableDataModal';
    modal.className = 'sb-table-data-modal';
    modal.hidden = true;

    modal.innerHTML = ''
        + '<div class="sb-table-data-modal__backdrop" data-table-action="close-data-modal"></div>'
        + '<div class="sb-table-data-modal__dialog">'
        + '  <div class="sb-table-data-modal__head">'
        + '      <div>'
        + '          <h2 class="sb-table-data-modal__title">Заполнение таблицы</h2>'
        + '          <p class="sb-table-data-modal__subtitle">Редактируй названия столбцов и значения строк в удобном виде</p>'
        + '      </div>'
        + '      <button class="sb-table-data-modal__close" type="button" data-table-action="close-data-modal">×</button>'
        + '  </div>'
        + '  <div class="sb-table-data-modal__toolbar">'
        + '      <button class="sb-btn sb-btn-light" type="button" data-table-action="modal-add-column">+ Столбец</button>'
        + '      <button class="sb-btn sb-btn-light" type="button" data-table-action="modal-add-row">+ Строка</button>'
        + '  </div>'
        + '  <div class="sb-table-data-modal__body" data-role="table-data-body"></div>'
        + '  <div class="sb-table-data-modal__footer">'
        + '      <button class="sb-btn sb-btn-light" type="button" data-table-action="close-data-modal">Отмена</button>'
        + '      <button class="sb-btn sb-btn-primary" type="button" data-table-action="apply-data-modal">Применить</button>'
        + '  </div>'
        + '</div>';

    document.body.appendChild(modal);

    return modal;
}

function renderTableDataModal() {
    var modal = ensureTableDataModal();
    var body = modal.querySelector('[data-role="table-data-body"]');

    if (!body) {
        return;
    }

    tableEditorDraft = normalizeTableContent(tableEditorDraft || {});

    var columns = tableEditorDraft.columns;
    var rows = tableEditorDraft.rows;

    var html = ''
        + '<div class="sb-table-data-scroll">'
        + '<table class="sb-table-data-grid">'
        + '  <thead>'
        + '      <tr>'
        + '          <th class="sb-table-data-grid__num">#</th>';

    columns.forEach(function (column) {
        html += ''
            + '<th data-table-modal-column-id="' + escapeHtml(column.id) + '">'
            + '  <div class="sb-table-data-column-head">'
            + '      <input class="sb-input sb-table-data-column-input" type="text" value="' + escapeHtml(column.label) + '" placeholder="Название столбца">'
            + '      <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-table-action="modal-delete-column" data-column-id="' + escapeHtml(column.id) + '">×</button>'
            + '  </div>'
            + '</th>';
    });

    html += ''
        + '      </tr>'
        + '  </thead>'
        + '  <tbody>';

    if (!rows.length) {
        html += ''
            + '<tr>'
            + '  <td colspan="' + (columns.length + 1) + '" class="sb-table-data-empty">Строк пока нет. Нажми “+ Строка”.</td>'
            + '</tr>';
    } else {
        rows.forEach(function (row, rowIndex) {
            var cells = row.cells || {};

            html += ''
                + '<tr data-table-modal-row-id="' + escapeHtml(row.id) + '">'
                + '  <td class="sb-table-data-grid__num">'
                + '      <div class="sb-table-data-row-num">'
                + '          <span>' + (rowIndex + 1) + '</span>'
                + '          <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-table-action="modal-delete-row" data-row-id="' + escapeHtml(row.id) + '">×</button>'
                + '      </div>'
                + '  </td>';

            columns.forEach(function (column) {
                html += ''
                    + '<td>'
                    + '  <textarea class="sb-table-data-cell" data-column-id="' + escapeHtml(column.id) + '" rows="2">' + escapeHtml(cells[column.id] || '') + '</textarea>'
                    + '</td>';
            });

            html += '</tr>';
        });
    }

    html += ''
        + '  </tbody>'
        + '</table>'
        + '</div>';

    body.innerHTML = html;
}

function collectTableDataFromModal() {
    var modal = ensureTableDataModal();

    var columns = [];
    var rows = [];

    modal.querySelectorAll('[data-table-modal-column-id]').forEach(function (columnNode, index) {
        var oldId = String(columnNode.getAttribute('data-table-modal-column-id') || '').trim();
        var input = columnNode.querySelector('.sb-table-data-column-input');
        var label = input ? String(input.value || '').trim() : '';

        if (!oldId) {
            oldId = 'col_' + (index + 1);
        }

        if (!label) {
            label = 'Столбец ' + (index + 1);
        }

        columns.push({
            id: oldId,
            label: label
        });
    });

    if (!columns.length) {
        columns = [
            {id: 'col_1', label: 'Столбец 1'}
        ];
    }

    modal.querySelectorAll('[data-table-modal-row-id]').forEach(function (rowNode, rowIndex) {
        var rowId = String(rowNode.getAttribute('data-table-modal-row-id') || '').trim();

        if (!rowId) {
            rowId = 'row_' + (Date.now() + rowIndex);
        }

        var cells = {};

        columns.forEach(function (column) {
            var input = rowNode.querySelector('[data-column-id="' + column.id + '"]');
            cells[column.id] = input ? String(input.value || '') : '';
        });

        rows.push({
            id: rowId,
            cells: cells
        });
    });

    var current = collectTableContentFromEditor();

    return {
        title: current.title || 'Таблица',
        columns: columns,
        rows: rows
    };
}

function openTableDataModal() {
    tableEditorDraft = collectTableContentFromEditor();

    var modal = ensureTableDataModal();
    modal.hidden = false;

    renderTableDataModal();
}

function closeTableDataModal() {
    var modal = ensureTableDataModal();
    modal.hidden = true;
}

function applyTableDataModal() {
    tableEditorDraft = collectTableDataFromModal();

    renderTableEditor(tableEditorDraft);
    closeTableDataModal();

    alert('Данные применены. Теперь нажми “Сохранить блок”, чтобы записать изменения.');
}

function addTableColumn() {
    var current = collectTableContentFromEditor();
    var newId = 'col_' + Date.now();

    current.columns.push({
        id: newId,
        label: 'Столбец ' + (current.columns.length + 1)
    });

    current.rows = current.rows.map(function (row) {
        row.cells = row.cells || {};
        row.cells[newId] = '';
        return row;
    });

    tableEditorDraft = current;
    renderTableEditor(tableEditorDraft);
}

function deleteTableColumn(columnId) {
    var current = collectTableContentFromEditor();

    if (current.columns.length <= 1) {
        alert('Нельзя удалить последний столбец');
        return;
    }

    current.columns = current.columns.filter(function (column) {
        return column.id !== columnId;
    });

    current.rows = current.rows.map(function (row) {
        if (row.cells) {
            delete row.cells[columnId];
        }

        return row;
    });

    tableEditorDraft = current;
    renderTableEditor(tableEditorDraft);
}

function addTableRow() {
    var current = collectTableContentFromEditor();
    var cells = {};

    current.columns.forEach(function (column) {
        cells[column.id] = '';
    });

    current.rows.push({
        id: 'row_' + Date.now(),
        cells: cells
    });

    tableEditorDraft = current;
    renderTableEditor(tableEditorDraft);
}

function deleteTableRow(rowId) {
    var current = collectTableContentFromEditor();

    current.rows = current.rows.filter(function (row) {
        return row.id !== rowId;
    });

    tableEditorDraft = current;
    renderTableEditor(tableEditorDraft);
}

function addTableColumnInModal() {
    tableEditorDraft = collectTableDataFromModal();

    var newId = 'col_' + Date.now();

    tableEditorDraft.columns.push({
        id: newId,
        label: 'Столбец ' + tableEditorDraft.columns.length
    });

    tableEditorDraft.rows = tableEditorDraft.rows.map(function (row) {
        row.cells = row.cells || {};
        row.cells[newId] = '';
        return row;
    });

    renderTableDataModal();
}

function addTableRowInModal() {
    tableEditorDraft = collectTableDataFromModal();

    var cells = {};

    tableEditorDraft.columns.forEach(function (column) {
        cells[column.id] = '';
    });

    tableEditorDraft.rows.push({
        id: 'row_' + Date.now(),
        cells: cells
    });

    renderTableDataModal();
}

function deleteTableColumnInModal(columnId) {
    tableEditorDraft = collectTableDataFromModal();

    if (tableEditorDraft.columns.length <= 1) {
        alert('Нельзя удалить последний столбец');
        return;
    }

    tableEditorDraft.columns = tableEditorDraft.columns.filter(function (column) {
        return column.id !== columnId;
    });

    tableEditorDraft.rows = tableEditorDraft.rows.map(function (row) {
        if (row.cells) {
            delete row.cells[columnId];
        }

        return row;
    });

    renderTableDataModal();
}

function deleteTableRowInModal(rowId) {
    tableEditorDraft = collectTableDataFromModal();

    tableEditorDraft.rows = tableEditorDraft.rows.filter(function (row) {
        return row.id !== rowId;
    });

    renderTableDataModal();
}

Важно: строка function fillVisualBlockForm(block) { должна остаться ниже этого блока.


---

2. В 60-events.js замени верхний обработчик таблицы

Файл:

/local/sitebuilder/assets/admin/editor/60-events.js

Если ты уже вставлял обработчик data-table-action, замени его целиком на этот:

document.addEventListener('click', function (e) {
    var tableAction = e.target.closest('[data-table-action]');

    if (!tableAction) {
        return;
    }

    var action = tableAction.getAttribute('data-table-action');

    if (action === 'open-data-modal') {
        openTableDataModal();
        return;
    }

    if (action === 'close-data-modal') {
        closeTableDataModal();
        return;
    }

    if (action === 'apply-data-modal') {
        applyTableDataModal();
        return;
    }

    if (action === 'add-column') {
        addTableColumn();
        return;
    }

    if (action === 'add-row') {
        addTableRow();
        return;
    }

    if (action === 'delete-column') {
        var columnNode = tableAction.closest('[data-table-column-id]');
        var columnId = columnNode ? String(columnNode.getAttribute('data-table-column-id') || '') : '';

        if (columnId) {
            deleteTableColumn(columnId);
        }

        return;
    }

    if (action === 'delete-row') {
        var rowNode = tableAction.closest('[data-table-row-id]');
        var rowId = rowNode ? String(rowNode.getAttribute('data-table-row-id') || '') : '';

        if (rowId) {
            deleteTableRow(rowId);
        }

        return;
    }

    if (action === 'modal-add-column') {
        addTableColumnInModal();
        return;
    }

    if (action === 'modal-add-row') {
        addTableRowInModal();
        return;
    }

    if (action === 'modal-delete-column') {
        var modalColumnId = String(tableAction.getAttribute('data-column-id') || '');

        if (modalColumnId) {
            deleteTableColumnInModal(modalColumnId);
        }

        return;
    }

    if (action === 'modal-delete-row') {
        var modalRowId = String(tableAction.getAttribute('data-row-id') || '');

        if (modalRowId) {
            deleteTableRowInModal(modalRowId);
        }

        return;
    }
});


---

3. В editor.css добавь стили большого окна

Файл:

/local/sitebuilder/assets/admin/editor.css

В самый конец добавь:

/* =========================================================
   Big table data modal
   ========================================================= */

.sb-table-editor__summary {
    display: grid;
    gap: 10px;
    padding: 12px;
    border: 1px solid #e2e8f0;
    border-radius: 14px;
    background: #fff;
}

.sb-table-editor__chips {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
}

.sb-table-editor__chip {
    display: inline-flex;
    align-items: center;
    min-height: 24px;
    padding: 3px 8px;
    border-radius: 999px;
    background: #eef2ff;
    color: #3730a3;
    font-size: 12px;
    font-weight: 800;
}

.sb-table-data-modal[hidden] {
    display: none !important;
}

.sb-table-data-modal {
    position: fixed;
    inset: 0;
    z-index: 99999;
}

.sb-table-data-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(15, 23, 42, .48);
}

.sb-table-data-modal__dialog {
    position: absolute;
    inset: 32px;
    display: flex;
    flex-direction: column;
    min-width: 0;
    min-height: 0;
    border-radius: 22px;
    background: #fff;
    box-shadow: 0 24px 80px rgba(15, 23, 42, .35);
    overflow: hidden;
}

.sb-table-data-modal__head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 20px;
    padding: 18px 22px;
    border-bottom: 1px solid #e2e8f0;
}

.sb-table-data-modal__title {
    margin: 0;
    color: #0f172a;
    font-size: 22px;
    font-weight: 900;
}

.sb-table-data-modal__subtitle {
    margin: 4px 0 0;
    color: #64748b;
    font-size: 13px;
}

.sb-table-data-modal__close {
    width: 38px;
    height: 38px;
    border: 0;
    border-radius: 12px;
    background: #f1f5f9;
    color: #0f172a;
    font-size: 26px;
    line-height: 1;
    cursor: pointer;
}

.sb-table-data-modal__toolbar {
    display: flex;
    gap: 10px;
    padding: 14px 22px;
    border-bottom: 1px solid #e2e8f0;
    background: #f8fafc;
}

.sb-table-data-modal__body {
    flex: 1;
    min-height: 0;
    padding: 18px 22px;
    overflow: auto;
}

.sb-table-data-scroll {
    width: 100%;
    overflow: auto;
}

.sb-table-data-grid {
    width: 100%;
    min-width: 900px;
    border-collapse: separate;
    border-spacing: 0;
    table-layout: fixed;
}

.sb-table-data-grid th,
.sb-table-data-grid td {
    padding: 8px;
    border-right: 1px solid #e2e8f0;
    border-bottom: 1px solid #e2e8f0;
    background: #fff;
    vertical-align: top;
}

.sb-table-data-grid th {
    position: sticky;
    top: 0;
    z-index: 2;
    background: #f8fafc;
}

.sb-table-data-grid th:first-child,
.sb-table-data-grid td:first-child {
    border-left: 1px solid #e2e8f0;
}

.sb-table-data-grid thead th {
    border-top: 1px solid #e2e8f0;
}

.sb-table-data-grid__num {
    width: 84px;
    min-width: 84px;
    max-width: 84px;
}

.sb-table-data-column-head {
    display: grid;
    grid-template-columns: minmax(0, 1fr) 34px;
    gap: 8px;
    align-items: center;
}

.sb-table-data-row-num {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 6px;
    min-height: 38px;
    color: #475569;
    font-size: 12px;
    font-weight: 900;
}

.sb-table-data-cell {
    width: 100%;
    min-height: 46px;
    resize: vertical;
    box-sizing: border-box;
    padding: 9px 10px;
    border: 1px solid #cbd5e1;
    border-radius: 10px;
    background: #fff;
    color: #0f172a;
    font: inherit;
    font-size: 13px;
    line-height: 1.35;
}

.sb-table-data-cell:focus,
.sb-table-data-column-input:focus {
    outline: none;
    border-color: #2563eb;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12);
}

.sb-table-data-empty {
    padding: 28px !important;
    color: #64748b;
    text-align: center;
}

.sb-table-data-modal__footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 22px;
    border-top: 1px solid #e2e8f0;
    background: #f8fafc;
}

@media (max-width: 900px) {
    .sb-table-data-modal__dialog {
        inset: 10px;
        border-radius: 16px;
    }
}


---

4. Обнови версии в editor.php

Поставь новые версии:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/30-blocks.js?v=6"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/60-events.js?v=6"></script>

И CSS:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.css?v=6">

Потом сделай Ctrl + F5.

Теперь в правой панели будет не куча маленьких полей, а кнопка “Открыть удобное заполнение”. Там можно будет нормально заполнять строки и столбцы в большом окне.