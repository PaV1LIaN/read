Похоже, браузер или сервер всё ещё берёт старый вариант, где дефолтные 3 столбца. Сделаем жёстче: после создания таблицы сразу явно сохраняем контент с нужным количеством столбцов через block.update.

В файле:

/local/sitebuilder/assets/admin/editor/30-blocks.js

найди функцию:

async function createBlock(type) {

и замени всю функцию целиком на эту:

async function createBlock(type) {
    if (!state.currentPageId) {
        alert('Сначала выберите страницу');
        return;
    }

    var content = {};
    var props = {};
    var isTableBlock = false;

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
        isTableBlock = true;

        var tableTitle = window.prompt('Название таблицы', 'Таблица');

        if (tableTitle === null) {
            return;
        }

        tableTitle = String(tableTitle || '').trim();

        if (!tableTitle) {
            tableTitle = 'Таблица';
        }

        var columnsCountRaw = window.prompt('Сколько столбцов создать?', '3');

        if (columnsCountRaw === null) {
            return;
        }

        columnsCountRaw = String(columnsCountRaw || '').replace(',', '.').trim();

        var columnsCount = parseInt(columnsCountRaw, 10);

        if (!columnsCount || isNaN(columnsCount) || columnsCount < 1) {
            columnsCount = 3;
        }

        if (columnsCount > 12) {
            columnsCount = 12;
        }

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
            title: tableTitle,
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

    if (createdBlockId > 0) {
        if (targetSectionId > 0) {
            await assignBlockToSection(createdBlockId, targetSectionId, targetColumn);
        }

        if (isTableBlock) {
            await api('block.update', {
                id: createdBlockId,
                content: JSON.stringify(content),
                props: JSON.stringify(props)
            });
        }

        state.currentBlockId = createdBlockId;
        await loadBlocks();
    }
}

Потом в editor.php обнови версию:

/assets/admin/editor/30-blocks.js?v=5

И сделай Ctrl + F5.

После этого при вводе 5 должно создаться ровно 5 столбцов: Столбец 1 … Столбец 5.