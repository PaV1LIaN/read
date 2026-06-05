Да, правильно. Тогда не зашиваем контакты в компонент. Контакты — только пример использования.

Компонент должен быть универсальным:

Таблица
├── заголовок таблицы
├── любое количество столбцов
├── свои названия столбцов
└── любое количество строк

Например им можно сделать:

Контакты: ФИО | Почта | Телефон
Прайс: Услуга | Цена | Срок
Сотрудники: ФИО | Должность | Отдел
Расписание: День | Время | Кабинет

Что меняем в предыдущем коде

1. В editor.php кнопку лучше сделать универсальной

Было:

<button class="sb-editor-add-card" type="button" data-add-block="table">
    <span class="sb-editor-add-card__title">Таблица</span>
    <span class="sb-editor-add-card__text">Столбцы и строки, например контакты</span>
</button>

Сделай так:

<button class="sb-editor-add-card" type="button" data-add-block="table">
    <span class="sb-editor-add-card__title">Таблица</span>
    <span class="sb-editor-add-card__text">Свои столбцы и строки для любых данных</span>
</button>


---

2. В форме таблицы текст тоже делаем универсальным

В tableBlockForm замени подсказки на такие:

<div id="tableBlockForm" class="sb-block-type-form" style="margin-top:12px;">
    <div class="sb-field">
        <label for="tableTitleInput">Заголовок таблицы</label>
        <input class="sb-input" type="text" id="tableTitleInput" placeholder="Например: Прайс-лист, контакты, расписание">
    </div>

    <div class="sb-table-editor" style="margin-top:12px;">
        <div class="sb-table-editor__head">
            <div>
                <strong>Столбцы</strong>
                <p class="sb-editor-note">Задай любое количество столбцов и назови их как нужно</p>
            </div>

            <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-table-action="add-column">
                + Столбец
            </button>
        </div>

        <div id="tableColumnsEditor" class="sb-table-editor__columns"></div>

        <div class="sb-table-editor__head" style="margin-top:16px;">
            <div>
                <strong>Строки</strong>
                <p class="sb-editor-note">Добавляй строки и заполняй значения по столбцам</p>
            </div>

            <button class="sb-btn sb-btn-primary sb-btn-small" type="button" data-table-action="add-row">
                + Строка
            </button>
        </div>

        <div id="tableRowsEditor" class="sb-table-editor__rows"></div>
    </div>
</div>


---

3. В 30-blocks.js сделай универсальные столбцы

Найди функцию:

function normalizeTableContent(content) {

И замени её целиком на эту:

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


---

4. В collectTableContentFromEditor() замени дефолтные столбцы

Найди внутри неё:

if (!columns.length) {
    columns = [
        {id: 'fio', label: 'ФИО'},
        {id: 'email', label: 'Почта'},
        {id: 'phone', label: 'Телефон'}
    ];
}

Замени на:

if (!columns.length) {
    columns = [
        {id: 'col_1', label: 'Столбец 1'},
        {id: 'col_2', label: 'Столбец 2'},
        {id: 'col_3', label: 'Столбец 3'}
    ];
}


---

5. В createBlock(type) сделай выбор начального количества столбцов

Найди кусок:

} else if (type === 'table') {
    content = {
        title: 'Контакты',
        columns: [
            {id: 'fio', label: 'ФИО'},
            {id: 'email', label: 'Почта'},
            {id: 'phone', label: 'Телефон'}
        ],
        rows: [
            {
                id: 'row_1',
                cells: {
                    fio: 'Иванов Иван',
                    email: 'ivanov@example.ru',
                    phone: '+7 999 000-00-00'
                }
            }
        ]
    };
} else if (type === 'disk') {

Замени на универсальный вариант:

} else if (type === 'table') {
    var columnsCountRaw = window.prompt('Сколько столбцов создать?', '3');
    var columnsCount = Number(columnsCountRaw || 3);

    if (!Number.isFinite(columnsCount) || columnsCount < 1) {
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

Теперь при добавлении компонента он спросит:

Сколько столбцов создать?

А потом пользователь сможет назвать их как угодно.


---

6. В addTableColumn() сделай универсальное название

Найди:

current.columns.push({
    id: newId,
    label: 'Новый столбец'
});

Можно оставить, но лучше так:

current.columns.push({
    id: newId,
    label: 'Столбец ' + (current.columns.length + 1)
});


---

7. Публичный шаблон table.php оставляем универсальным

Файл:

/local/sitebuilder/views/blocks/table.php

Его логика уже универсальная. Он не должен знать про контакты, почту или телефон. Он просто выводит любые columns и любые rows.


---

После этих замен компонент будет универсальным: при создании выбираешь начальное количество столбцов, потом называешь столбцы как нужно и добавляешь строки.