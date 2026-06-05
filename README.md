Да, сделаем нормально: при создании таблицы будем спрашивать название таблицы, а потом его можно будет менять в правой панели в поле “Заголовок таблицы” и нажимать “Сохранить блок”.

1. В 30-blocks.js замени кусок создания table

Файл:

/local/sitebuilder/assets/admin/editor/30-blocks.js

Найди внутри createBlock(type) вот этот кусок:

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

Замени на:

} else if (type === 'table') {
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
        title: tableTitle,
        columns: tableColumns,
        rows: [
            {
                id: 'row_1',
                cells: tableCells
            }
        ]
    };


---

2. Проверь, что в editor.php есть поле названия

В форме таблицы должно быть:

<div class="sb-field">
    <label for="tableTitleInput">Заголовок таблицы</label>
    <input class="sb-input" type="text" id="tableTitleInput" placeholder="Например: Прайс-лист, контакты, расписание">
</div>

Если этого поля нет — вставь его внутрь:

<div id="tableBlockForm" class="sb-block-type-form" style="margin-top:12px;">

самым первым элементом.


---

3. Как менять название уже созданной таблицы

1. Нажми на блок таблицы в редакторе.


2. Справа появится поле “Заголовок таблицы”.


3. Введи новое название.


4. Нажми “Сохранить блок”.




---

4. Обнови версию файла

В editor.php поменяй:

/assets/admin/editor/30-blocks.js?v=3

на:

/assets/admin/editor/30-blocks.js?v=4

Потом Ctrl + F5.