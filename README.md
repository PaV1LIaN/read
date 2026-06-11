Да, это уже понятная ошибка.

Когда ты меняешь тип данных, у нас сейчас вызывается:

renderTableFromContent(typeRoot);

То есть JS полностью перерисовывает таблицу. В какой-то момент он собирает columns пустыми, и поэтому остаётся только служебный столбец №.

Нужно сделать правильно: при смене типа данных не перерисовывать всю таблицу, а менять только ячейки этого конкретного столбца.


---

1. В table-edit.js добавь защиту от пустых columns

Файл:

/local/sitebuilder/assets/public/table-edit.js

Найди функцию:

function renderTableFromContent(root) {

Внутри неё найди:

var columns = Array.isArray(content.columns) ? content.columns : [];
var rows = Array.isArray(content.rows) ? content.rows : [];
var hasControlCol = true;

Сразу после этого добавь:

if (!columns.length) {
    console.warn('SiteBuilder table: render stopped because columns is empty');
    return;
}

Должно стать так:

var columns = Array.isArray(content.columns) ? content.columns : [];
var rows = Array.isArray(content.rows) ? content.rows : [];
var hasControlCol = true;

if (!columns.length) {
    console.warn('SiteBuilder table: render stopped because columns is empty');
    return;
}


---

2. Добавь функцию смены типа столбца

В table-edit.js найди функцию:

function renderCellEditor(td, column, row, content) {

После всей этой функции вставь:

function changeColumnType(root, th, newType) {
    if (!root || !th) {
        return;
    }

    newType = normalizeType(newType);

    var columnId = String(th.getAttribute('data-column-id') || '');

    if (!columnId) {
        return;
    }

    th.setAttribute('data-column-type-value', newType);

    var typeSelect = th.querySelector('[data-column-type]');

    if (typeSelect) {
        typeSelect.value = newType;
    }

    var formulaInput = th.querySelector('[data-column-formula]');

    if (formulaInput) {
        formulaInput.style.display = newType === 'formula' ? '' : 'none';
    }

    var formulaTools = th.querySelector('[data-formula-tools]');

    if (formulaTools) {
        formulaTools.style.display = newType === 'formula' ? '' : 'none';
    }

    var content = collectContentFromDom(root);
    var columns = Array.isArray(content.columns) ? content.columns : [];
    var rows = Array.isArray(content.rows) ? content.rows : [];

    var column = columns.find(function (item) {
        return String(item.id || '') === columnId;
    });

    if (!column) {
        return;
    }

    column.type = newType;

    rows.forEach(function (row, rowIndex) {
        row.cells = row.cells || {};

        var tr = root.querySelector('tbody tr[data-row-id="' + cssEscape(row.id) + '"]');

        if (!tr) {
            return;
        }

        var td = tr.querySelector('td[data-column-id="' + cssEscape(columnId) + '"]');

        if (!td) {
            td = document.createElement('td');
            tr.appendChild(td);
        }

        renderCellEditor(td, column, row, content);
    });

    setContent(root, collectContentFromDom(root));
    applyWidths(root);
    applyAllAligns(root);
    updateFormulaCells(root);
    setDirty(root, true);
}


---

3. Замени обработчик смены типа

В table-edit.js найди блок:

var typeSelect = e.target.closest('[data-column-type]');

if (typeSelect) {
    var typeRoot = typeSelect.closest('[data-public-editable-table]');

    if (!typeRoot) {
        return;
    }

    e.stopImmediatePropagation();

    var typeContent = collectContentFromDom(typeRoot);

    setContent(typeRoot, typeContent);
    renderTableFromContent(typeRoot);
    setDirty(typeRoot, true);
    return;
}

Полностью замени его на:

var typeSelect = e.target.closest('[data-column-type]');

if (typeSelect) {
    var typeRoot = typeSelect.closest('[data-public-editable-table]');
    var typeTh = typeSelect.closest('th[data-column-id]');

    if (!typeRoot || !typeTh) {
        return;
    }

    e.stopImmediatePropagation();

    changeColumnType(typeRoot, typeTh, typeSelect.value);
    return;
}


---

4. Почему так произошло

Сейчас при смене типа было так:

1. выбрал тип “Число”;
2. JS собрал данные;
3. JS полностью очистил thead/tbody;
4. потом попытался заново построить таблицу;
5. columns где-то стал пустым;
6. остался только №.

После правки будет так:

1. выбрал тип “Число”;
2. JS нашёл только этот столбец;
3. заменил только ячейки этого столбца на input для числа;
4. остальные столбцы не трогает.


---

5. Ещё одна ошибка, которую видно по скрину

У тебя сейчас в шапке показывается:

Код:c1
Код:c2
Код:c3

Это нормально, но лучше визуально сделать с пробелом. В CSS у нас уже было:

.sb-public-table-column-code::before {
    content: "Код: ";
}

Значит в самом HTML/JS внутри textContent должно быть только:

c1

А не:

Код:c1

Если у тебя после правок вдруг будет показываться Код: Код:c1, значит где-то в JS/PHP в textContent попал уже готовый текст с “Код:”. Тогда скажи, поправим.

После этих трёх правок поменяй тип данных ещё раз. Таблица не должна схлопываться до одного №.