Да, я вижу главную ошибку.

В edit=Y формулу считает JS, он уже понимает c1 + c2.

А без edit=Y формулу считает PHP в table.php, и он пока ищет только реальные id столбцов, типа:

c_1
c_2
col_178065...

Поэтому в режиме редактирования получается 110, а на обычной странице становится 0.

Нужно сделать одно правило:

Пользователь видит и пишет в формуле c1, c2, c3.
Внутри таблицы могут быть любые id, но у каждого столбца есть короткий code.
PHP и JS оба понимают code.


---

1. Правка table.php

Файл:

/local/sitebuilder/views/blocks/table.php

1.1. Добавь функцию короткого кода

После функции:

$normalizeType = static function ($type): string {

после её закрытия добавь:

$normalizeColumnCode = static function (array $column, int $index): string {
    $code = trim((string)($column['code'] ?? ''));

    if ($code === '') {
        $id = trim((string)($column['id'] ?? ''));

        if (preg_match('/^c_?(\d+)$/i', $id, $m)) {
            $code = 'c' . $m[1];
        } else {
            $code = 'c' . ($index + 1);
        }
    }

    if (preg_match('/^c_(\d+)$/i', $code, $m)) {
        $code = 'c' . $m[1];
    }

    $code = preg_replace('/[^A-Za-z0-9_]/', '', $code);

    if ($code === '') {
        $code = 'c' . ($index + 1);
    }

    return $code;
};


---

1.2. В нормализации колонок добавь code

Найди:

$columns = array_values(array_map(static function ($column, $index) use ($normalizeAlign, $normalizeType) {

Замени на:

$columns = array_values(array_map(static function ($column, $index) use ($normalizeAlign, $normalizeType, $normalizeColumnCode) {

Ниже в return [ добавь code.

Было примерно так:

return [
    'id' => $id,
    'label' => $label,
    'width' => $width,
    'align' => $normalizeAlign($column['align'] ?? 'left'),
    'type' => $normalizeType($column['type'] ?? 'text'),
    'formula' => trim((string)($column['formula'] ?? '')),
];

Замени на:

return [
    'id' => $id,
    'code' => $normalizeColumnCode($column, $index),
    'label' => $label,
    'width' => $width,
    'align' => $normalizeAlign($column['align'] ?? 'left'),
    'type' => $normalizeType($column['type'] ?? 'text'),
    'formula' => trim((string)($column['formula'] ?? '')),
];


---

1.3. Замени функцию $calculateFormula

Найди старую функцию:

$calculateFormula = static function (string $formula, array $row) use ($valueToNumber, $evalMathExpression): string {

И замени её полностью на:

$calculateFormula = static function (string $formula, array $row, array $columns) use ($valueToNumber, $evalMathExpression): string {
    $formula = trim($formula);

    if ($formula === '') {
        return '';
    }

    $cells = is_array($row['cells'] ?? null) ? $row['cells'] : [];

    $tokenMap = [];

    foreach ($columns as $index => $column) {
        $id = (string)($column['id'] ?? '');
        $code = (string)($column['code'] ?? ('c' . ($index + 1)));

        if ($id !== '') {
            $tokenMap[$id] = $id;
        }

        if ($code !== '') {
            $tokenMap[$code] = $id;
        }

        if (preg_match('/^c(\d+)$/i', $code, $m)) {
            $tokenMap['c_' . $m[1]] = $id;
        }

        if (preg_match('/^c_(\d+)$/i', $id, $m)) {
            $tokenMap['c' . $m[1]] = $id;
        }
    }

    $expression = preg_replace_callback('/\b[A-Za-z_][A-Za-z0-9_]*\b/', static function ($matches) use ($cells, $tokenMap, $valueToNumber) {
        $token = $matches[0];

        if (!isset($tokenMap[$token])) {
            return '0';
        }

        $columnId = $tokenMap[$token];

        return (string)$valueToNumber($cells[$columnId] ?? '');
    }, $formula);

    return (string)$evalMathExpression((string)$expression);
};


---

1.4. Исправь вызовы $calculateFormula

Найди:

$renderViewCell = static function (array $column, array $row) use ($valueToText, $normalizeDate, $calculateFormula): string {

Замени на:

$renderViewCell = static function (array $column, array $row) use ($valueToText, $normalizeDate, $calculateFormula, $columns): string {

Внутри найди:

return sb_public_h($calculateFormula((string)($column['formula'] ?? ''), $row));

Замени на:

return sb_public_h($calculateFormula((string)($column['formula'] ?? ''), $row, $columns));

Ниже в edit-режиме найди:

<?= sb_public_h($calculateFormula($column['formula'], $row)) ?>

Замени на:

<?= sb_public_h($calculateFormula($column['formula'], $row, $columns)) ?>


---

1.5. В шапке показывай code, а не id

Найди:

<span class="sb-public-table-column-code"><?= sb_public_h($column['id']) ?></span>

Замени на:

<span class="sb-public-table-column-code" data-column-code><?= sb_public_h($column['code']) ?></span>

После этого PHP уже будет понимать формулу:

c1 + c2

и без edit=Y.


---

2. Правка table-edit.js

Файл:

/local/sitebuilder/assets/public/table-edit.js

2.1. Добавь функцию нормализации кода

После функции:

function normalizeType(type) {

после её закрытия добавь:

function normalizeColumnCode(code, index, id) {
    code = String(code || '').trim();
    id = String(id || '').trim();

    if (!code) {
        var idMatch = id.match(/^c_?(\d+)$/i);

        if (idMatch) {
            code = 'c' + idMatch[1];
        } else {
            code = 'c' + (index + 1);
        }
    }

    var codeMatch = code.match(/^c_(\d+)$/i);

    if (codeMatch) {
        code = 'c' + codeMatch[1];
    }

    code = code.replace(/[^A-Za-z0-9_]/g, '');

    if (!code) {
        code = 'c' + (index + 1);
    }

    return code;
}


---

2.2. В collectContentFromDom() сохраняй code

Найди внутри collectContentFromDom() место:

var oldColumn = findOldColumn(oldContent, columnId);
var width = oldColumn && oldColumn.width
    ? Number(oldColumn.width)
    : getColumnCurrentWidth(table, columnId);

columns.push({
    id: columnId,
    label: label,
    width: clampWidth(width),
    align: getColumnAlignFromTh(th),
    type: getColumnTypeFromTh(th),
    formula: getColumnFormulaFromTh(th)
});

Замени на:

var oldColumn = findOldColumn(oldContent, columnId);
var width = oldColumn && oldColumn.width
    ? Number(oldColumn.width)
    : getColumnCurrentWidth(table, columnId);

var codeNode = th.querySelector('[data-column-code]');
var codeFromDom = codeNode ? textValue(codeNode).replace(/^Код:\s*/i, '') : '';
var oldCode = oldColumn && oldColumn.code ? String(oldColumn.code) : '';
var columnCode = normalizeColumnCode(oldCode || codeFromDom, index, columnId);

columns.push({
    id: columnId,
    code: columnCode,
    label: label,
    width: clampWidth(width),
    align: getColumnAlignFromTh(th),
    type: getColumnTypeFromTh(th),
    formula: getColumnFormulaFromTh(th)
});


---

2.3. Замени calculateFormula()

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
        var column = null;

        columns.some(function (item, index) {
            var id = String(item.id || '');
            var code = normalizeColumnCode(item.code || '', index, id);
            var legacyCode = code.replace(/^c(\d+)$/i, 'c_$1');

            if (token === id || token === code || token === legacyCode) {
                column = item;
                return true;
            }

            return false;
        });

        if (!column) {
            return '0';
        }

        return String(valueToNumber(cells[column.id]));
    });

    return evaluateMathExpression(expression);
}


---

2.4. В renderTableFromContent() показывай нормальный код

Найди:

columns.forEach(function (column) {

Внутри renderTableFromContent() замени на:

columns.forEach(function (column, columnIndex) {

Ниже найди:

var code = document.createElement('span');
code.className = 'sb-public-table-column-code';
code.textContent = column.id;

Замени на:

column.code = normalizeColumnCode(column.code || '', columnIndex, column.id);

var code = document.createElement('span');
code.className = 'sb-public-table-column-code';
code.setAttribute('data-column-code', '');
code.textContent = column.code;


---

2.5. В помощнике формулы тоже используй code

Внутри создания formulaColumnSelect найди:

columns.forEach(function (item) {

Замени на:

columns.forEach(function (item, itemIndex) {

Ниже замени:

option.value = item.code || item.id;
option.textContent = (item.code || item.id) + ' — ' + (item.label || 'Столбец');

на:

var optionCode = normalizeColumnCode(item.code || '', itemIndex, item.id);

option.value = optionCode;
option.textContent = optionCode + ' — ' + (item.label || 'Столбец');


---

2.6. В addColumn() оставь code

Проверь, чтобы в addColumn() было так:

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

3. Почему на обычной странице было 0

Потому что формула была:

c1 + c2

Но PHP искал ячейки с ключами:

c1
c2

А реальные ключи в строке были:

c_1
c_2

или длинные col_....

Теперь будет так:

c1 → находим настоящий id столбца → берём значение из cells[настоящий id]

То есть пользователь пишет коротко:

c1 + c2

а код сам понимает, где лежат данные.

После правок открой таблицу в edit=Y, нажми Сохранить изменения, потом открой без edit=Y. Формула должна остаться 110, а не 0.