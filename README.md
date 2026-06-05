Да, делаем следующий этап: редактирование таблицы прямо на public-странице.

Что появится в режиме:

/public.php?siteId=13&pageId=14&edit=Y

Можно будет:

1. менять название таблицы;
2. менять названия столбцов прямо в шапке;
3. менять значения ячеек прямо в таблице;
4. добавлять строки;
5. удалять строки;
6. менять ширину столбцов;
7. сохранять всё одной кнопкой.


---

1. Замени table.php

Файл:

/local/sitebuilder/views/blocks/table.php

Полностью замени на:

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

$columns = array_values(array_map(static function ($column, $index) {
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
                        $style = '';

                        if ((int)$column['width'] > 0) {
                            $style = ' style="width:' . (int)$column['width'] . 'px;"';
                        }
                        ?>
                        <th data-column-id="<?= sb_public_h($column['id']) ?>"<?= $style ?>>
                            <span
                                class="sb-public-table__th-text"
                                <?php if ($isEditMode): ?>
                                    contenteditable="true"
                                    data-column-label
                                <?php endif; ?>
                            ><?= sb_public_h($column['label']) ?></span>

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
    window.SB_TABLE_EDIT_LOADED = 'v5-inline-edit';

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
                width: clampWidth(width)
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
   Public table inline edit
   ========================================================= */

.sb-public-table-editbar__main {
    min-width: 240px;
    flex: 1;
}

.sb-public-table-editbar__label {
    display: grid;
    gap: 5px;
    color: #1e40af;
    font-size: 12px;
    font-weight: 900;
}

.sb-public-table-title-input {
    width: 100%;
    min-height: 36px;
    box-sizing: border-box;
    padding: 7px 10px;
    border: 1px solid rgba(37, 99, 235, .35);
    border-radius: 10px;
    background: #fff;
    color: #0f172a;
    font: inherit;
    font-size: 14px;
    font-weight: 800;
}

.sb-public-table-editbar__actions {
    display: flex;
    align-items: center;
    gap: 8px;
}

.sb-public-table-editbar__btn--light {
    background: #fff !important;
    color: #1e40af !important;
    border: 1px solid rgba(37, 99, 235, .35) !important;
}

.sb-public-table__control-col {
    width: 72px !important;
}

.sb-public-table__control-th,
.sb-public-table__control-td {
    width: 72px !important;
    min-width: 72px !important;
    max-width: 72px !important;
    text-align: center !important;
    background: #f8fafc !important;
}

.sb-public-table-row-actions {
    display: flex;
    align-items: center;
    justify-content: center;
    gap: 6px;
}

.sb-public-table-row-num {
    min-width: 20px;
    color: #64748b;
    font-size: 12px;
    font-weight: 900;
}

.sb-public-table-row-delete {
    width: 24px;
    height: 24px;
    border: 0;
    border-radius: 8px;
    background: #fee2e2;
    color: #991b1b;
    font-size: 16px;
    font-weight: 900;
    line-height: 1;
    cursor: pointer;
}

.sb-public-table-row-delete:hover {
    background: #fecaca;
}

.sb-public-table--editable [contenteditable="true"] {
    outline: none;
    cursor: text;
}

.sb-public-table--editable [contenteditable="true"]:focus {
    box-shadow: inset 0 0 0 2px rgba(37, 99, 235, .45);
    background: #eff6ff;
}

.sb-public-table--editable td[data-cell-editable] {
    white-space: pre-wrap;
}

.sb-public-table--editable .sb-public-table__th-text {
    min-height: 22px;
    border-radius: 8px;
    padding: 3px 8px 3px 3px;
}

.sb-public-table--editable .sb-public-table__th-text:focus {
    background: #eff6ff;
}


---

4. Обнови версию подключения

В файле:

/local/sitebuilder/views/layout/public_page.php

Поставь:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js?v=11"></script>

И версию CSS:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=17">


---

После Ctrl + F5 проверь в консоли:

window.SB_TABLE_EDIT_LOADED

Должно быть:

"v5-inline-edit"

Теперь можно кликать прямо в ячейки и заголовки, менять текст, добавлять/удалять строки и нажимать “Сохранить изменения”.