Да, по скрину видно 3 проблемы:

1. Коды столбцов огромные: col_1780654750135 — неудобно для формул.
2. Формулу руками писать неудобно.
3. Поля внутри ячеек слишком короткие, должны растягиваться на ширину столбца.

Делаем так:

1. Для пользователя показываем короткие коды: c1, c2, c3.
2. Формула будет работать и с короткими кодами: c1 * c2.
3. В столбце типа “Формула” появится выбор столбца и кнопка “Вставить”.
4. Поля ввода в ячейках растянем на 100% ширины столбца.

1. Добавь CSS

В конец файла:

/local/sitebuilder/assets/public/public.css

добавь:

/* =========================================================
   Public table UX improvements
   ========================================================= */

.sb-public-table th {
    vertical-align: top;
}

.sb-public-table-th-inner {
    display: grid;
    gap: 6px;
    min-width: 0;
}

.sb-public-table-column-code {
    display: inline-flex;
    width: fit-content;
    max-width: 100%;
    padding: 2px 7px;
    border-radius: 999px;
    background: #e2e8f0;
    color: #475569;
    font-size: 10px;
    font-weight: 900;
    line-height: 1.3;
}

.sb-public-table-column-code::before {
    content: "Код: ";
    opacity: .75;
}

.sb-public-table-cell-input,
.sb-public-table-cell-link input,
.sb-public-table-cell-image input {
    display: block;
    width: 100% !important;
    min-width: 100% !important;
    max-width: 100% !important;
    min-height: 32px;
    box-sizing: border-box;
}

.sb-public-table--editable td {
    min-width: 0;
}

.sb-public-table--editable td[data-column-id] {
    padding: 8px;
}

.sb-public-table-formula-tools {
    display: grid;
    gap: 5px;
    padding: 6px;
    border-radius: 10px;
    background: #f8fafc;
    border: 1px solid rgba(148, 163, 184, .45);
}

.sb-public-table-formula-tools__row {
    display: flex;
    gap: 5px;
}

.sb-public-table-formula-tools select,
.sb-public-table-formula-tools button {
    min-height: 26px;
    box-sizing: border-box;
    border-radius: 8px;
    font-size: 11px;
    font-weight: 800;
}

.sb-public-table-formula-tools select {
    width: 100%;
    min-width: 0;
    border: 1px solid rgba(148, 163, 184, .7);
    background: #fff;
    color: #334155;
}

.sb-public-table-formula-tools button {
    border: 0;
    padding: 3px 7px;
    background: #dbeafe;
    color: #1e40af;
    cursor: pointer;
}

.sb-public-table-formula-tools button:hover {
    background: #bfdbfe;
}


---

2. В table-edit.js добавь короткие коды c1, c2, c3

Файл:

/local/sitebuilder/assets/public/table-edit.js

Найди функцию:

function collectContentFromDom(root) {

Внутри неё найди место, где в columns.push сейчас добавляется столбец:

columns.push({
    id: columnId,
    label: label,
    width: clampWidth(width),
    align: getColumnAlignFromTh(th),
    type: getColumnTypeFromTh(th),
    formula: getColumnFormulaFromTh(th)
});

Замени на:

var oldCode = oldColumn && oldColumn.code ? String(oldColumn.code) : '';

columns.push({
    id: columnId,
    code: oldCode || ('c' + (index + 1)),
    label: label,
    width: clampWidth(width),
    align: getColumnAlignFromTh(th),
    type: getColumnTypeFromTh(th),
    formula: getColumnFormulaFromTh(th)
});


---

3. Формулы должны понимать c1, c2, c3

Найди функцию:

function calculateFormula(content, row, formula) {

Полностью замени её на:

function calculateFormula(content, row, formula) {
    formula = String(formula || '').trim();

    if (!formula) {
        return '';
    }

    var cells = row && row.cells ? row.cells : {};
    var columns = Array.isArray(content.columns) ? content.columns : [];

    var expression = formula.replace(/\b[A-Za-z_][A-Za-z0-9_]*\b/g, function (token) {
        var column = columns.find(function (item) {
            return String(item.id || '') === token || String(item.code || '') === token;
        });

        if (!column) {
            return '0';
        }

        return String(valueToNumber(cells[column.id]));
    });

    return evaluateMathExpression(expression);
}

Теперь формула:

c1 * c2

будет работать так же, как:

col_1780654750135 * col_1781155812339


---

4. В шапке показываем короткий код

Найди в renderTableFromContent(root) кусок:

var code = document.createElement('span');
code.className = 'sb-public-table-column-code';
code.textContent = column.id;

Замени на:

var code = document.createElement('span');
code.className = 'sb-public-table-column-code';
code.textContent = column.code || column.id;


---

5. При добавлении столбца сохраняем code

Найди в addColumn(root):

columns.push({
    id: newColumnId,
    label: 'Столбец ' + newIndex,
    width: 160,
    align: 'left',
    type: 'text',
    formula: ''
});

Замени на:

columns.push({
    id: newColumnId,
    code: 'c' + newIndex,
    label: 'Столбец ' + newIndex,
    width: 160,
    align: 'left',
    type: 'text',
    formula: ''
});


---

6. Добавь выбор столбца для формулы

Найди в renderTableFromContent(root) кусок:

var formulaInput = document.createElement('input');
formulaInput.className = 'sb-public-table-formula-input';
formulaInput.type = 'text';
formulaInput.placeholder = 'Например: col_1 * col_2';
formulaInput.value = column.formula || '';
formulaInput.setAttribute('data-column-formula', '');

if (column.type !== 'formula') {
    formulaInput.style.display = 'none';
}

Сразу после него вставь:

var formulaTools = document.createElement('div');
formulaTools.className = 'sb-public-table-formula-tools';
formulaTools.setAttribute('data-formula-tools', '');

if (column.type !== 'formula') {
    formulaTools.style.display = 'none';
}

var formulaColumnSelect = document.createElement('select');
formulaColumnSelect.setAttribute('data-formula-insert-column', '');

columns.forEach(function (item) {
    if (item.id === column.id) {
        return;
    }

    var option = document.createElement('option');
    option.value = item.code || item.id;
    option.textContent = (item.code || item.id) + ' — ' + (item.label || 'Столбец');

    formulaColumnSelect.appendChild(option);
});

var formulaInsertBtn = document.createElement('button');
formulaInsertBtn.type = 'button';
formulaInsertBtn.setAttribute('data-formula-insert-btn', '');
formulaInsertBtn.textContent = 'Вставить';

var formulaOps = document.createElement('div');
formulaOps.className = 'sb-public-table-formula-tools__row';

['+', '-', '*', '/', '(', ')'].forEach(function (op) {
    var btn = document.createElement('button');
    btn.type = 'button';
    btn.setAttribute('data-formula-op', op);
    btn.textContent = op;
    formulaOps.appendChild(btn);
});

var formulaInsertRow = document.createElement('div');
formulaInsertRow.className = 'sb-public-table-formula-tools__row';
formulaInsertRow.appendChild(formulaColumnSelect);
formulaInsertRow.appendChild(formulaInsertBtn);

formulaTools.appendChild(formulaInsertRow);
formulaTools.appendChild(formulaOps);

Потом ниже найди:

inner.appendChild(typeSelect);
inner.appendChild(formulaInput);
inner.appendChild(alignSelect);

Замени на:

inner.appendChild(typeSelect);
inner.appendChild(formulaInput);
inner.appendChild(formulaTools);
inner.appendChild(alignSelect);


---

7. При смене типа показываем/скрываем помощник формулы

В обработчике change найди часть:

if (typeSelect) {

Внутри она у тебя вызывает renderTableFromContent(typeRoot);. Это нормально, потому что при смене типа таблица перерисуется уже с помощником формулы.


---

8. Добавь обработчик вставки в формулу

В table-edit.js найди обработчик:

document.addEventListener('click', function (e) {

В самое начало этого обработчика вставь:

var formulaInsertBtn = e.target.closest('[data-formula-insert-btn]');

if (formulaInsertBtn) {
    var formulaRoot = formulaInsertBtn.closest('[data-public-editable-table]');
    var formulaTh = formulaInsertBtn.closest('th[data-column-id]');
    var formulaInput = formulaTh ? formulaTh.querySelector('[data-column-formula]') : null;
    var formulaSelect = formulaTh ? formulaTh.querySelector('[data-formula-insert-column]') : null;

    if (formulaRoot && formulaInput && formulaSelect && formulaSelect.value) {
        e.preventDefault();
        e.stopImmediatePropagation();

        formulaInput.value = String(formulaInput.value || '').trim();

        if (formulaInput.value !== '') {
            formulaInput.value += ' ';
        }

        formulaInput.value += formulaSelect.value;

        setContent(formulaRoot, collectContentFromDom(formulaRoot));
        updateFormulaCells(formulaRoot);
        setDirty(formulaRoot, true);
    }

    return;
}

var formulaOpBtn = e.target.closest('[data-formula-op]');

if (formulaOpBtn) {
    var opRoot = formulaOpBtn.closest('[data-public-editable-table]');
    var opTh = formulaOpBtn.closest('th[data-column-id]');
    var opInput = opTh ? opTh.querySelector('[data-column-formula]') : null;
    var op = formulaOpBtn.getAttribute('data-formula-op') || '';

    if (opRoot && opInput && op) {
        e.preventDefault();
        e.stopImmediatePropagation();

        opInput.value = String(opInput.value || '').trim();

        if (opInput.value !== '' && op !== ')') {
            opInput.value += ' ';
        }

        opInput.value += op;

        if (op !== '(') {
            opInput.value += ' ';
        }

        setContent(opRoot, collectContentFromDom(opRoot));
        updateFormulaCells(opRoot);
        setDirty(opRoot, true);
    }

    return;
}


---

9. Добавь исключение для клика мышкой

В mousedown-обработчике найди список исключений:

e.target.closest('[data-date-cell]')

После него добавь:

||
e.target.closest('[data-formula-insert-column]') ||
e.target.closest('[data-formula-insert-btn]') ||
e.target.closest('[data-formula-op]')

Чтобы кнопки формулы не конфликтовали с изменением ширины столбца.


---

После этого на экране будет так:

Столбец 1
Код: c1
Тип: Число

Столбец 2
Код: c2
Тип: Число

Столбец 3
Код: c3
Тип: Формула
Формула: c1 * c2

И формулу можно будет собирать кнопками, не вводя длинные col_1780654750135.