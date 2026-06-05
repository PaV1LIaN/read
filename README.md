Да, в твоём случае проще заменить весь 30-blocks.js целиком, чтобы не ловить, куда именно вставлять куски.

Файл:

/local/sitebuilder/assets/admin/editor/30-blocks.js

Замени полностью на это:

function blockPreviewText(block) {
    var type = String(block.type || '');
    var content = block.content || {};
    var props = block.props || {};
    var sectionId = getBlockSectionId(block);
    var column = getBlockColumn(block);

    var placementText = sectionId > 0 ? ' · секция #' + sectionId + ' · колонка ' + column : '';

    if (type === 'heading') {
        return (content.text || '[пустой заголовок]') + placementText;
    }

    if (type === 'text') {
        return (content.text || '[пустой текст]') + placementText;
    }

    if (type === 'button') {
        return (content.label || 'Кнопка') + (content.href ? ' → ' + content.href : '') + placementText;
    }

    if (type === 'html') {
        return ((content.html || '').slice(0, 220) || '[пустой HTML]') + placementText;
    }

    if (type === 'table') {
        var columnsCount = Array.isArray(content.columns) ? content.columns.length : 0;
        var rowsCount = Array.isArray(content.rows) ? content.rows.length : 0;

        return 'Таблица: '
            + (content.title || 'Без названия')
            + ' · столбцов: ' + columnsCount
            + ' · строк: ' + rowsCount
            + placementText;
    }

    if (type === 'disk') {
        return 'Компонент "Диск": '
            + (props.title || 'Файлы')
            + ' · rootMode=' + (props.rootMode || 'site')
            + ' · view=' + (props.viewMode || 'table')
            + placementText;
    }

    try {
        return JSON.stringify(content) + placementText;
    } catch (e) {
        return '[контент блока]' + placementText;
    }
}

function renderBlocks() {
    if (!blocksList) {
        return;
    }

    if (!state.currentPageId) {
        blocksList.innerHTML = ''
            + '<div class="sb-editor-empty-big">'
            + '   <strong>Страница не выбрана</strong>'
            + '   Выбери страницу слева, чтобы редактировать блоки'
            + '</div>';
        return;
    }

    if (!state.pageSections.length) {
        if (!state.blocks.length) {
            blocksList.innerHTML = ''
                + '<div class="sb-editor-empty-big">'
                + '   <strong>На странице пока нет блоков</strong>'
                + '   Добавь первый блок через панель сверху'
                + '</div>';
            return;
        }

        blocksList.innerHTML = state.blocks.map(function (block) {
            var active = Number(block.id || 0) === state.currentBlockId ? ' is-active' : '';

            return ''
                + '<div class="sb-editor-block' + active + '" draggable="true" data-block-id="' + Number(block.id || 0) + '">'
                + '  <div class="sb-editor-block-head">'
                + '      <div>'
                + '          <h3 class="sb-editor-block-title">' + escapeHtml(block.type || 'block') + '</h3>'
                + '          <div class="sb-editor-chip">block #' + Number(block.id || 0) + '</div>'
                + '      </div>'
                + '  </div>'
                + '  <div class="sb-editor-block-preview">' + escapeHtml(blockPreviewText(block)) + '</div>'
                + '</div>';
        }).join('');

        return;
    }

    var grouped = groupBlocksBySectionAndColumn();

    blocksList.innerHTML = state.pageSections.map(function (section) {
        var sectionId = Number(section.id || 0);
        var layout = section.layout || {};
        var columns = getSectionColumns(sectionId);
        var activeSection = Number(state.currentSectionId || 0) === sectionId ? ' is-active' : '';

        var html = ''
            + '<div class="sb-editor-section-preview' + activeSection + '" data-editor-section-id="' + sectionId + '">'
            + '  <div class="sb-editor-section-preview__head" data-page-section-select="' + sectionId + '">'
            + '      <div>'
            + '          <h3 class="sb-editor-section-preview__title">' + escapeHtml(section.title || 'Секция') + '</h3>'
            + '          <div class="sb-editor-section-preview__meta">'
            + '              <span>' + columns + ' кол.</span>'
            + '              <span>' + escapeHtml(layout.container || 'default') + '</span>'
            + '          </div>'
            + '      </div>'
            + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-add-block-to-section="' + sectionId + '">Выбрать</button>'
            + '  </div>'
            + '  <div class="sb-editor-section-preview__grid sb-editor-section-preview__grid--' + columns + '">';

        for (var column = 1; column <= columns; column++) {
            var blocks = grouped[sectionId] && grouped[sectionId][column]
                ? grouped[sectionId][column]
                : [];

            var isTargetColumn =
                Number(state.currentSectionId || 0) === sectionId &&
                Number(state.currentColumn || 1) === column;

            html += ''
                + '<div class="sb-editor-section-preview__column' + (isTargetColumn ? ' is-target' : '') + '" data-section-id="' + sectionId + '" data-column="' + column + '">'
                + '  <div class="sb-editor-section-preview__column-head">'
                + '      <div class="sb-editor-section-preview__column-title">Колонка ' + column + '</div>'
                + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-set-add-target="' + sectionId + '" data-column="' + column + '">'
                +          (isTargetColumn ? 'Выбрано' : 'Добавлять сюда')
                + '      </button>'
                + '  </div>';

            if (!blocks.length) {
                html += '<div class="sb-editor-section-preview__empty">Пусто</div>';
            } else {
                html += blocks.map(function (block) {
                    var active = Number(block.id || 0) === state.currentBlockId ? ' is-active' : '';

                    return ''
                        + '<div class="sb-editor-block' + active + '" draggable="true" data-block-id="' + Number(block.id || 0) + '">'
                        + '  <div class="sb-editor-block-head">'
                        + '      <div>'
                        + '          <h3 class="sb-editor-block-title">' + escapeHtml(block.type || 'block') + '</h3>'
                        + '          <div class="sb-editor-chip">block #' + Number(block.id || 0) + '</div>'
                        + '      </div>'
                        + '  </div>'
                        + '  <div class="sb-editor-block-preview">' + escapeHtml(blockPreviewText(block)) + '</div>'
                        + '</div>';
                }).join('');
            }

            html += '</div>';
        }

        html += ''
            + '  </div>'
            + '</div>';

        return html;
    }).join('');
}

function hideAllBlockTypeForms() {
    [
        'headingBlockForm',
        'textBlockForm',
        'buttonBlockForm',
        'htmlBlockForm',
        'tableBlockForm',
        'diskBlockForm',
        'unknownBlockForm'
    ].forEach(function (id) {
        var node = document.getElementById(id);
        if (node) {
            node.classList.remove('is-active');
            node.classList.add('sb-hidden');
        }
    });
}

function showBlockTypeForm(id) {
    var node = document.getElementById(id);
    if (!node) return;

    node.classList.add('is-active');
    node.classList.remove('sb-hidden');
}

/* =========================================================
   TABLE BLOCK
   ========================================================= */

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
    content = normalizeTableContent(content);

    var titleInput = document.getElementById('tableTitleInput');
    var columnsNode = document.getElementById('tableColumnsEditor');
    var rowsNode = document.getElementById('tableRowsEditor');

    if (titleInput) {
        titleInput.value = content.title || '';
    }

    if (columnsNode) {
        columnsNode.innerHTML = content.columns.map(function (column, index) {
            return ''
                + '<div class="sb-table-editor__column" data-table-column-id="' + escapeHtml(column.id) + '">'
                + '  <span class="sb-table-editor__column-index">' + (index + 1) + '</span>'
                + '  <input class="sb-input sb-table-column-label" type="text" value="' + escapeHtml(column.label) + '" placeholder="Название столбца">'
                + '  <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-table-action="delete-column">×</button>'
                + '</div>';
        }).join('');
    }

    if (rowsNode) {
        rowsNode.innerHTML = content.rows.map(function (row, rowIndex) {
            var cells = row.cells || {};

            var rowHtml = ''
                + '<div class="sb-table-editor__row" data-table-row-id="' + escapeHtml(row.id) + '">'
                + '  <div class="sb-table-editor__row-head">'
                + '      <strong>Строка ' + (rowIndex + 1) + '</strong>'
                + '      <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-table-action="delete-row">Удалить</button>'
                + '  </div>'
                + '  <div class="sb-table-editor__row-cells" style="grid-template-columns: repeat(' + content.columns.length + ', minmax(0, 1fr));">';

            content.columns.forEach(function (column) {
                rowHtml += ''
                    + '<label class="sb-table-editor__cell">'
                    + '  <span>' + escapeHtml(column.label) + '</span>'
                    + '  <input class="sb-input sb-table-cell-input" type="text" data-column-id="' + escapeHtml(column.id) + '" value="' + escapeHtml(cells[column.id] || '') + '">'
                    + '</label>';
            });

            rowHtml += ''
                + '  </div>'
                + '</div>';

            return rowHtml;
        }).join('');
    }
}

function collectTableContentFromEditor() {
    var titleInput = document.getElementById('tableTitleInput');
    var columnsNode = document.getElementById('tableColumnsEditor');
    var rowsNode = document.getElementById('tableRowsEditor');

    var columns = [];
    var rows = [];

    if (columnsNode) {
        columnsNode.querySelectorAll('[data-table-column-id]').forEach(function (columnNode, index) {
            var oldId = String(columnNode.getAttribute('data-table-column-id') || '').trim();
            var labelInput = columnNode.querySelector('.sb-table-column-label');
            var label = labelInput ? String(labelInput.value || '').trim() : '';

            if (!label) {
                label = 'Столбец ' + (index + 1);
            }

            columns.push({
                id: oldId || ('col_' + (Date.now() + index)),
                label: label
            });
        });
    }

    if (!columns.length) {
        columns = [
            {id: 'col_1', label: 'Столбец 1'},
            {id: 'col_2', label: 'Столбец 2'},
            {id: 'col_3', label: 'Столбец 3'}
        ];
    }

    if (rowsNode) {
        rowsNode.querySelectorAll('[data-table-row-id]').forEach(function (rowNode, rowIndex) {
            var rowId = String(rowNode.getAttribute('data-table-row-id') || '').trim();

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
    }

    return {
        title: titleInput ? String(titleInput.value || '').trim() : '',
        columns: columns,
        rows: rows
    };
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

    renderTableEditor(current);
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

    renderTableEditor(current);
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

    renderTableEditor(current);
}

function deleteTableRow(rowId) {
    var current = collectTableContentFromEditor();

    current.rows = current.rows.filter(function (row) {
        return row.id !== rowId;
    });

    renderTableEditor(current);
}

function fillVisualBlockForm(block) {
    hideAllBlockTypeForms();

    var type = String(block.type || '');
    var content = block.content || {};

    if (type === 'heading') {
        showBlockTypeForm('headingBlockForm');

        var headingTextInput = document.getElementById('headingTextInput');
        if (headingTextInput) {
            headingTextInput.value = content.text || '';
        }

        return;
    }

    if (type === 'text') {
        showBlockTypeForm('textBlockForm');

        var textTextInput = document.getElementById('textTextInput');
        if (textTextInput) {
            textTextInput.value = content.text || '';
        }

        return;
    }

    if (type === 'button') {
        showBlockTypeForm('buttonBlockForm');

        var buttonLabelInput = document.getElementById('buttonLabelInput');
        var buttonHrefInput = document.getElementById('buttonHrefInput');
        var buttonTargetInput = document.getElementById('buttonTargetInput');

        if (buttonLabelInput) {
            buttonLabelInput.value = content.label || '';
        }

        if (buttonHrefInput) {
            buttonHrefInput.value = content.href || '';
        }

        if (buttonTargetInput) {
            buttonTargetInput.value = content.target || '_self';
        }

        return;
    }

    if (type === 'html') {
        showBlockTypeForm('htmlBlockForm');

        var htmlInput = document.getElementById('htmlInput');
        if (htmlInput) {
            htmlInput.value = content.html || '';
        }

        return;
    }

    if (type === 'table') {
        showBlockTypeForm('tableBlockForm');
        renderTableEditor(content);
        return;
    }

    if (type === 'disk') {
        showBlockTypeForm('diskBlockForm');
        return;
    }

    showBlockTypeForm('unknownBlockForm');

    var jsonFields = document.getElementById('blockJsonFields');
    if (jsonFields) {
        jsonFields.classList.add('is-open');
    }
}

function fillDiskForm(props) {
    props = props || {};

    var diskTitleInput = document.getElementById('diskTitleInput');
    var diskRootModeInput = document.getElementById('diskRootModeInput');
    var diskViewModeInput = document.getElementById('diskViewModeInput');
    var diskPermissionModeInput = document.getElementById('diskPermissionModeInput');
    var diskMaxFileSizeInput = document.getElementById('diskMaxFileSizeInput');
    var diskAllowedExtensionsInput = document.getElementById('diskAllowedExtensionsInput');

    if (diskTitleInput) {
        diskTitleInput.value = props.title || 'Файлы';
    }

    if (diskRootModeInput) {
        diskRootModeInput.value = props.rootMode || 'site';
    }

    if (diskViewModeInput) {
        diskViewModeInput.value = props.viewMode || 'table';
    }

    if (diskPermissionModeInput) {
        diskPermissionModeInput.value = props.permissionMode || 'inherit_site';
    }

    if (diskMaxFileSizeInput) {
        diskMaxFileSizeInput.value = props.maxFileSize || 52428800;
    }

    if (diskAllowedExtensionsInput) {
        diskAllowedExtensionsInput.value = Array.isArray(props.allowedExtensions) ? props.allowedExtensions.join(' ') : '';
    }

    var checks = {
        diskAllowUploadInput: !!props.allowUpload,
        diskAllowCreateFolderInput: !!props.allowCreateFolder,
        diskAllowRenameInput: !!props.allowRename,
        diskAllowDeleteInput: !!props.allowDelete,
        diskAllowDownloadInput: !!props.allowDownload,
        diskShowSearchInput: !!props.showSearch,
        diskShowBreadcrumbsInput: !!props.showBreadcrumbs,
        diskUseSiteRootFallbackInput: !!props.useSiteRootFallback
    };

    Object.keys(checks).forEach(function (id) {
        var node = document.getElementById(id);
        if (node) {
            node.checked = checks[id];
        }
    });
}

function fillBlockForm() {
    var block = getCurrentBlock();
    var emptyNode = document.getElementById('blockInspectorEmpty');
    var formNode = document.getElementById('blockInspector');

    if (!block) {
        if (emptyNode) {
            emptyNode.classList.remove('sb-hidden');
        }

        if (formNode) {
            formNode.classList.add('sb-hidden');
        }

        hideAllBlockTypeForms();

        var blockTypeInput = document.getElementById('blockTypeInput');
        var blockContentInput = document.getElementById('blockContentInput');
        var blockPropsInput = document.getElementById('blockPropsInput');

        if (blockTypeInput) {
            blockTypeInput.value = '';
        }

        if (blockContentInput) {
            blockContentInput.value = '';
        }

        if (blockPropsInput) {
            blockPropsInput.value = '';
        }

        var jsonFieldsEmpty = document.getElementById('blockJsonFields');
        if (jsonFieldsEmpty) {
            jsonFieldsEmpty.classList.remove('is-open');
        }

        fillBlockPlacementForm(null);

        return;
    }

    if (emptyNode) {
        emptyNode.classList.add('sb-hidden');
    }

    if (formNode) {
        formNode.classList.remove('sb-hidden');
    }

    var content = block.content || {};
    var props = block.props || {};

    var blockTypeInputFilled = document.getElementById('blockTypeInput');
    var blockContentInputFilled = document.getElementById('blockContentInput');
    var blockPropsInputFilled = document.getElementById('blockPropsInput');

    if (blockTypeInputFilled) {
        blockTypeInputFilled.value = block.type || '';
    }

    if (blockContentInputFilled) {
        blockContentInputFilled.value = JSON.stringify(content, null, 2);
    }

    if (blockPropsInputFilled) {
        blockPropsInputFilled.value = JSON.stringify(props, null, 2);
    }

    var jsonFields = document.getElementById('blockJsonFields');
    if (jsonFields) {
        jsonFields.classList.remove('is-open');
    }

    if (block.type === 'disk') {
        fillDiskForm(props);
    }

    fillVisualBlockForm(block);
    fillBlockPlacementForm(block);
}

function collectDiskBlockProps(block) {
    var oldProps = block.props || {};

    return {
        title: getInputValue('diskTitleInput').trim() || 'Файлы',
        rootMode: getInputValue('diskRootModeInput') || 'site',
        rootFolderId: oldProps.rootFolderId || null,
        viewMode: getInputValue('diskViewModeInput') || 'table',
        permissionMode: getInputValue('diskPermissionModeInput') || 'inherit_site',
        maxFileSize: Number(getInputValue('diskMaxFileSizeInput') || 0),
        allowedExtensions: String(getInputValue('diskAllowedExtensionsInput') || '')
            .trim()
            .split(/\s+/)
            .filter(Boolean),
        allowUpload: getChecked('diskAllowUploadInput'),
        allowCreateFolder: getChecked('diskAllowCreateFolderInput'),
        allowRename: getChecked('diskAllowRenameInput'),
        allowDelete: getChecked('diskAllowDeleteInput'),
        allowDownload: getChecked('diskAllowDownloadInput'),
        showSearch: getChecked('diskShowSearchInput'),
        showBreadcrumbs: getChecked('diskShowBreadcrumbsInput'),
        useSiteRootFallback: getChecked('diskUseSiteRootFallbackInput'),
        defaultSort: oldProps.defaultSort || 'updatedAt',
        defaultSortDirection: oldProps.defaultSortDirection || 'desc',

        sectionId: oldProps.sectionId || null,
        column: oldProps.column || null,
        _placement: oldProps._placement || null
    };
}

function collectVisualBlockData(block) {
    var type = String(block.type || '');
    var content = {};
    var props = block.props || {};

    if (type === 'heading') {
        return {
            content: {
                text: getInputValue('headingTextInput').trim()
            },
            props: props
        };
    }

    if (type === 'text') {
        return {
            content: {
                text: getInputValue('textTextInput')
            },
            props: props
        };
    }

    if (type === 'button') {
        return {
            content: {
                label: getInputValue('buttonLabelInput').trim() || 'Кнопка',
                href: getInputValue('buttonHrefInput').trim() || '#',
                target: getInputValue('buttonTargetInput') || '_self'
            },
            props: props
        };
    }

    if (type === 'html') {
        return {
            content: {
                html: getInputValue('htmlInput')
            },
            props: props
        };
    }

    if (type === 'table') {
        return {
            content: collectTableContentFromEditor(),
            props: props
        };
    }

    if (type === 'disk') {
        return {
            content: block.content || {},
            props: collectDiskBlockProps(block)
        };
    }

    try {
        content = JSON.parse(document.getElementById('blockContentInput').value || '{}');
    } catch (e) {
        alert('Контент блока должен быть валидным JSON');
        return null;
    }

    try {
        props = JSON.parse(document.getElementById('blockPropsInput').value || '{}');
    } catch (e) {
        alert('Свойства блока должны быть валидным JSON');
        return null;
    }

    return {
        content: content,
        props: props
    };
}

async function createBlock(type) {
    if (!state.currentPageId) {
        alert('Сначала выберите страницу');
        return;
    }

    var content = {};
    var props = {};

    if (type === 'heading') {
        content = {text: 'Новый заголовок'};
    } else if (type === 'text') {
        content = {text: 'Новый текстовый блок'};
    } else if (type === 'button') {
        content = {
            label: 'Кнопка',
            href: '#',
            target: '_self'
        };
    } else if (type === 'html') {
        content = {html: '<div>Новый HTML блок</div>'};
    } else if (type === 'table') {
        var columnsCountRaw = window.prompt('Сколько столбцов создать?', '3');
        var columnsCount = Number(columnsCountRaw || 3);

        if (!columnsCount || isNaN(columnsCount) || columnsCount < 1) {
            columnsCount = 3;
        }

        if (columnsCount > 12) {
            columnsCount = 12;
        }

        columnsCount = Math.floor(columnsCount);

        var tableColumns = [];
        var tableCells = {};

        for (var i = 1; i <= columnsCount; i++) {
            var columnId = 'col_' + i;

            tableColumns.push({
                id: columnId,
                label: 'Столбец ' + i
            });

            tableCells[columnId] = '';
        }

        content = {
            title: 'Таблица',
            columns: tableColumns,
            rows: [
                {
                    id: 'row_1',
                    cells: tableCells
                }
            ]
        };
    } else if (type === 'disk') {
        content = {};
        props = {
            title: 'Файлы',
            rootMode: 'site',
            rootFolderId: null,
            viewMode: 'table',
            allowUpload: true,
            allowCreateFolder: true,
            allowRename: true,
            allowDelete: true,
            allowDownload: true,
            showSearch: true,
            showBreadcrumbs: true,
            defaultSort: 'updatedAt',
            defaultSortDirection: 'desc',
            allowedExtensions: [],
            maxFileSize: 52428800,
            permissionMode: 'inherit_site',
            useSiteRootFallback: true
        };
    }

    var targetSectionId = getDefaultSectionId();
    var targetColumn = getDefaultColumn();

    props.sectionId = targetSectionId;
    props.column = targetColumn;
    props._placement = {
        sectionId: targetSectionId,
        column: targetColumn
    };

    var createRes = await api('block.create', {
        pageId: state.currentPageId,
        type: type,
        content: JSON.stringify(content),
        props: JSON.stringify(props),
        sectionId: targetSectionId,
        column: targetColumn
    });

    await loadBlocks();

    var createdBlockId = Number(
        (createRes.block && createRes.block.id) ||
        (createRes.data && createRes.data.block && createRes.data.block.id) ||
        0
    );

    if (!createdBlockId && state.blocks.length) {
        var sortedBlocks = state.blocks.slice().sort(function (a, b) {
            return Number(b.id || 0) - Number(a.id || 0);
        });

        createdBlockId = Number(sortedBlocks[0].id || 0);
    }

    if (createdBlockId > 0 && targetSectionId > 0) {
        await assignBlockToSection(createdBlockId, targetSectionId, targetColumn);
        state.currentBlockId = createdBlockId;
        await loadBlocks();
    }
}

async function saveBlock() {
    var block = getCurrentBlock();
    if (!block) return;

    var collected = collectVisualBlockData(block);

    if (!collected) {
        return;
    }

    await api('block.update', {
        id: block.id,
        content: JSON.stringify(collected.content),
        props: JSON.stringify(collected.props)
    });

    await saveBlockPlacement(block);

    await loadBlocks();
}

async function duplicateBlock() {
    var block = getCurrentBlock();
    if (!block) return;

    await api('block.duplicate', {
        id: block.id
    });

    await loadBlocks();
}

async function deleteBlock() {
    var block = getCurrentBlock();
    if (!block) return;
    if (!confirm('Удалить блок?')) return;

    await api('block.delete', {
        id: block.id
    });

    state.currentBlockId = 0;
    await loadBlocks();
}

async function moveBlock(dir) {
    var block = getCurrentBlock();
    if (!block) return;

    await api('block.move', {
        id: block.id,
        dir: dir
    });

    await loadBlocks();
}

После этого не забудь добавить обработчик в 60-events.js, иначе кнопки + Столбец, + Строка, удаления не будут работать.