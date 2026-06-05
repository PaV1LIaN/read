Да, сделаем так:

1. В public edit-mode появится кнопка “+ Столбец”.
2. У каждого столбца появится выбор выравнивания: слева / центр / справа.
3. Выравнивание будет сохраняться в content.columns[].align.
4. На обычной public-странице таблица тоже будет показываться с этим выравниванием.


---

1. Замени table.php

Файл:

/local/sitebuilder/views/blocks/table.php

Полностью замени на этот код:

<?php
global $USER;

$title = trim((string)($content['title'] ?? ''));

$columns = is_array($content['columns'] ?? null) ? $content['columns'] : [];
$rows = is_array($content['rows'] ?? null) ? $content['rows'] : [];

if (empty($columns)) {
    return;
}

$isEditMode = (
    (string)($_GET['edit'] ?? '') === 'Y'
    && is_object($USER)
    && method_exists($USER, 'IsAuthorized')
    && $USER->IsAuthorized()
    && method_exists($USER, 'IsAdmin')
    && $USER->IsAdmin()
);

$normalizeAlign = static function ($align): string {
    $align = (string)$align;

    if (!in_array($align, ['left', 'center', 'right'], true)) {
        return 'left';
    }

    return $align;
};

$columns = array_values(array_map(static function ($column, $index) use ($normalizeAlign) {
    $id = trim((string)($column['id'] ?? ''));

    if ($id === '') {
        $id = 'col_' . ($index + 1);
    }

    $label = trim((string)($column['label'] ?? ''));

    if ($label === '') {
        $label = 'Столбец ' . ($index + 1);
    }

    $width = (int)($column['width'] ?? 0);

    if ($width < 40) {
        $width = 0;
    }

    if ($width > 1200) {
        $width = 1200;
    }

    return [
        'id' => $id,
        'label' => $label,
        'width' => $width,
        'align' => $normalizeAlign($column['align'] ?? 'left'),
    ];
}, $columns, array_keys($columns)));

$normalizedRows = [];

foreach ($rows as $rowIndex => $row) {
    $cells = is_array($row['cells'] ?? null) ? $row['cells'] : [];
    $rowId = trim((string)($row['id'] ?? ''));

    if ($rowId === '') {
        $rowId = 'row_' . ($rowIndex + 1);
    }

    $normalizedRows[] = [
        'id' => $rowId,
        'cells' => $cells,
    ];
}

$tableContent = [
    'title' => $title !== '' ? $title : 'Таблица',
    'columns' => $columns,
    'rows' => $normalizedRows,
];

$contentJson = json_encode($tableContent, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
$propsJson = json_encode(is_array($props ?? null) ? $props : [], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

$blockId = (int)($block['id'] ?? 0);
?>

<section
    class="sb-block sb-block--table<?= $isEditMode ? ' is-public-editable-table' : '' ?>"
    <?php if ($isEditMode): ?>
        data-public-editable-table
        data-block-id="<?= $blockId ?>"
        data-content="<?= sb_public_h((string)$contentJson) ?>"
        data-props="<?= sb_public_h((string)$propsJson) ?>"
    <?php endif; ?>
>
    <?php if ($isEditMode): ?>
        <div class="sb-public-table-editbar">
            <div class="sb-public-table-editbar__main">
                <label class="sb-public-table-editbar__label">
                    Название таблицы
                    <input
                        class="sb-public-table-title-input"
                        type="text"
                        value="<?= sb_public_h($title !== '' ? $title : 'Таблица') ?>"
                        data-table-title-input
                    >
                </label>
            </div>

            <div class="sb-public-table-editbar__actions">
                <button class="sb-public-table-editbar__btn sb-public-table-editbar__btn--light" type="button" data-table-add-column>
                    + Столбец
                </button>

                <button class="sb-public-table-editbar__btn sb-public-table-editbar__btn--light" type="button" data-table-add-row>
                    + Строка
                </button>

                <button class="sb-public-table-editbar__btn" type="button" data-table-save-all>
                    Сохранить изменения
                </button>
            </div>
        </div>
    <?php else: ?>
        <?php if ($title !== ''): ?>
            <h2 class="sb-public-table__title"><?= sb_public_h($title) ?></h2>
        <?php endif; ?>
    <?php endif; ?>

    <div class="sb-public-table-wrap">
        <table class="sb-public-table<?= $isEditMode ? ' sb-public-table--editable' : '' ?>">
            <colgroup>
                <?php if ($isEditMode): ?>
                    <col class="sb-public-table__control-col" style="width:72px;">
                <?php endif; ?>

                <?php foreach ($columns as $column): ?>
                    <?php
                    $style = '';

                    if ((int)$column['width'] > 0) {
                        $style = ' style="width:' . (int)$column['width'] . 'px;"';
                    }
                    ?>
                    <col data-column-id="<?= sb_public_h($column['id']) ?>"<?= $style ?>>
                <?php endforeach; ?>
            </colgroup>

            <thead>
                <tr>
                    <?php if ($isEditMode): ?>
                        <th class="sb-public-table__control-th">№</th>
                    <?php endif; ?>

                    <?php foreach ($columns as $column): ?>
                        <?php
                        $styleParts = [
                            'text-align:' . $column['align'],
                        ];

                        if ((int)$column['width'] > 0) {
                            $styleParts[] = 'width:' . (int)$column['width'] . 'px';
                        }

                        $style = ' style="' . sb_public_h(implode(';', $styleParts)) . '"';
                        ?>
                        <th
                            data-column-id="<?= sb_public_h($column['id']) ?>"
                            data-column-align-value="<?= sb_public_h($column['align']) ?>"
                            <?= $style ?>
                        >
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
                                <?php endif; ?>
                            </div>

                            <?php if ($isEditMode): ?>
                                <span class="sb-public-table-resizer" data-column-resizer></span>
                            <?php endif; ?>
                        </th>
                    <?php endforeach; ?>
                </tr>
            </thead>

            <tbody>
                <?php if (!empty($normalizedRows)): ?>
                    <?php foreach ($normalizedRows as $rowIndex => $row): ?>
                        <?php $cells = is_array($row['cells'] ?? null) ? $row['cells'] : []; ?>

                        <tr data-row-id="<?= sb_public_h((string)$row['id']) ?>">
                            <?php if ($isEditMode): ?>
                                <td class="sb-public-table__control-td">
                                    <div class="sb-public-table-row-actions">
                                        <span class="sb-public-table-row-num"><?= $rowIndex + 1 ?></span>
                                        <button type="button" class="sb-public-table-row-delete" data-table-delete-row title="Удалить строку">×</button>
                                    </div>
                                </td>
                            <?php endif; ?>

                            <?php foreach ($columns as $column): ?>
                                <td
                                    data-column-id="<?= sb_public_h($column['id']) ?>"
                                    style="text-align:<?= sb_public_h($column['align']) ?>"
                                    <?php if ($isEditMode): ?>
                                        contenteditable="true"
                                        data-cell-editable
                                    <?php endif; ?>
                                ><?= nl2br(sb_public_h((string)($cells[$column['id']] ?? ''))) ?></td>
                            <?php endforeach; ?>
                        </tr>
                    <?php endforeach; ?>
                <?php else: ?>
                    <tr data-empty-row>
                        <td colspan="<?= count($columns) + ($isEditMode ? 1 : 0) ?>">Нет данных</td>
                    </tr>
                <?php endif; ?>
            </tbody>
        </table>
    </div>
</section>


---

2. Замени table-edit.js

Файл:

/local/sitebuilder/assets/public/table-edit.js

Полностью замени на:

(function () {
    window.SB_TABLE_EDIT_LOADED = 'v6-inline-columns-align';

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

        if (['left', 'center', 'right'].indexOf(align) === -1) {
            return 'left';
        }

        return align;
    }

    function textValue(node) {
        return String(node ? (node.innerText || node.textContent || '') : '').replace(/\u00a0/g, ' ').trim();
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

        var controlTd = document.createElement('td');
        controlTd.className = 'sb-public-table__control-td';
        controlTd.innerHTML = ''
            + '<div class="sb-public-table-row-actions">'
            + '  <span class="sb-public-table-row-num"></span>'
            + '  <button type="button" class="sb-public-table-row-delete" data-table-delete-row title="Удалить строку">×</button>'
            + '</div>';

        tr.appendChild(controlTd);

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
        applyWidths(root);
        applyAllAligns(root);
        renumberRows(root);

        root.addEventListener('input', function (e) {
            if (
                e.target.matches('[data-table-title-input]') ||
                e.target.matches('[data-column-label]') ||
                e.target.matches('[data-cell-editable]')
            ) {
                setContent(root, collectContentFromDom(root));
                setDirty(root, true);
            }
        });

        root.addEventListener('change', function (e) {
            if (e.target.matches('[data-column-align]')) {
                var th = e.target.closest('th[data-column-id]');
                var columnId = th ? String(th.getAttribute('data-column-id') || '') : '';
                var align = normalizeAlign(e.target.value);

                if (columnId) {
                    applyColumnAlign(root, columnId, align);
                    setContent(root, collectContentFromDom(root));
                    setDirty(root, true);
                }
            }
        });

        root.addEventListener('keydown', function (e) {
            if (e.target.matches('[data-cell-editable], [data-column-label]')) {
                if (e.key === 'Enter' && !e.shiftKey) {
                    e.preventDefault();
                    e.target.blur();
                }
            }
        });

        var saveBtn = root.querySelector('[data-table-save-all]');

        if (saveBtn) {
            saveBtn.addEventListener('click', function () {
                saveBlock(root);
            });
        }

        var addRowBtn = root.querySelector('[data-table-add-row]');

        if (addRowBtn) {
            addRowBtn.addEventListener('click', function () {
                addRow(root);
            });
        }

        var addColumnBtn = root.querySelector('[data-table-add-column]');

        if (addColumnBtn) {
            addColumnBtn.addEventListener('click', function () {
                addColumn(root);
            });
        }

        root.addEventListener('click', function (e) {
            var deleteBtn = e.target.closest('[data-table-delete-row]');

            if (deleteBtn) {
                deleteRow(root, deleteBtn);
            }
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

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initAllTables);
    } else {
        initAllTables();
    }
})();


---

3. Добавь CSS в public.css

Файл:

/local/sitebuilder/assets/public/public.css

В самый конец добавь:

/* =========================================================
   Public table columns and align
   ========================================================= */

.sb-public-table-th-inner {
    display: grid;
    gap: 6px;
    min-width: 0;
}

.sb-public-table-align-select {
    width: 100%;
    min-height: 28px;
    box-sizing: border-box;
    padding: 3px 7px;
    border: 1px solid rgba(148, 163, 184, .7);
    border-radius: 8px;
    background: #fff;
    color: #334155;
    font-size: 12px;
    font-weight: 800;
}

.sb-public-table-align-select:focus {
    outline: none;
    border-color: #2563eb;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12);
}

.sb-public-table-editbar__actions {
    flex-wrap: wrap;
}


---

4. Обнови версии подключения

В файле:

/local/sitebuilder/views/layout/public_page.php

Поставь:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js?v=12"></script>

И CSS:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=18">

После этого сделай Ctrl + F5.

Проверка в консоли:

window.SB_TABLE_EDIT_LOADED

Должно быть:

"v6-inline-columns-align"