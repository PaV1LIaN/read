В 60-events.js нужно добавить только обработчик кнопок таблицы.

Файл:

/local/sitebuilder/assets/admin/editor/60-events.js

Вставь в самый верх файла, до всего остального кода:

document.addEventListener('click', function (e) {
    var tableAction = e.target.closest('[data-table-action]');

    if (!tableAction) {
        return;
    }

    var action = tableAction.getAttribute('data-table-action');

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
});

То есть начало 60-events.js должно стать примерно таким:

document.addEventListener('click', function (e) {
    var tableAction = e.target.closest('[data-table-action]');

    if (!tableAction) {
        return;
    }

    var action = tableAction.getAttribute('data-table-action');

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
});

if (pagesList) {
    pagesList.addEventListener('click', async function (e) {
        // дальше твой старый код

После этого в editor.php обнови версию подключения:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/60-events.js?v=2"></script>

И сделай Ctrl + F5.