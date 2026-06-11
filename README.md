Да, начинаем этап 1: типы столбцов + гиперссылка + рисунок + формула.

Сейчас добавим:

text      — обычный текст
number    — число
date      — дата
link      — гиперссылка
image     — рисунок по URL
formula   — вычисляемый столбец

Формула пока простая, например:

col_1 * col_2


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

$normalizeAlign = static function ($align): string {
    $align = (string)$align;

    if (!in_array($align, ['left', 'center', 'right'], true)) {
        return 'left';
    }

    return $align;
};

$normalizeType = static function ($type): string {
    $type = (string)$type;

    if (!in_array($type, ['text', 'number', 'date', 'link', 'image', 'formula'], true)) {
        return 'text';
    }

    return $type;
};

$valueToText = static function ($value): string {
    if (is_array($value)) {
        foreach (['text', 'url', 'src', 'alt'] as $key) {
            if (isset($value[$key]) && trim((string)$value[$key]) !== '') {
                return trim((string)$value[$key]);
            }
        }

        return '';
    }

    return trim((string)$value);
};

$valueToNumber = static function ($value) use ($valueToText): float {
    $text = $valueToText($value);
    $text = str_replace([' ', ','], ['', '.'], $text);

    if (!is_numeric($text)) {
        return 0.0;
    }

    return (float)$text;
};

$evalMathExpression = static function (string $expression) {
    $expression = trim(str_replace(',', '.', $expression));

    if ($expression === '') {
        return '';
    }

    if (!preg_match('/^[0-9+\-*\/().\s]+$/', $expression)) {
        return '';
    }

    preg_match_all('/\d+(?:\.\d+)?|[+\-*\/()]/', $expression, $matches);

    $tokens = $matches[0] ?? [];

    if (empty($tokens)) {
        return '';
    }

    $raw = preg_replace('/\s+/', '', $expression);
    $joined = implode('', $tokens);

    if ($raw !== $joined) {
        return '';
    }

    $i = 0;
    $count = count($tokens);

    $parseExpression = null;
    $parseTerm = null;
    $parseFactor = null;

    $parseFactor = static function () use (&$tokens, &$i, &$count, &$parseExpression, &$parseFactor) {
        if ($i >= $count) {
            return 0.0;
        }

        $token = $tokens[$i];

        if ($token === '+') {
            $i++;
            return $parseFactor();
        }

        if ($token === '-') {
            $i++;
            return -$parseFactor();
        }

        if ($token === '(') {
            $i++;
            $value = $parseExpression();

            if ($i < $count && $tokens[$i] === ')') {
                $i++;
            }

            return $value;
        }

        $i++;

        return (float)$token;
    };

    $parseTerm = static function () use (&$tokens, &$i, &$count, &$parseFactor) {
        $value = $parseFactor();

        while ($i < $count && ($tokens[$i] === '*' || $tokens[$i] === '/')) {
            $op = $tokens[$i];
            $i++;
            $right = $parseFactor();

            if ($op === '*') {
                $value *= $right;
            } else {
                if (abs($right) < 0.0000001) {
                    return 0.0;
                }

                $value /= $right;
            }
        }

        return $value;
    };

    $parseExpression = static function () use (&$tokens, &$i, &$count, &$parseTerm) {
        $value = $parseTerm();

        while ($i < $count && ($tokens[$i] === '+' || $tokens[$i] === '-')) {
            $op = $tokens[$i];
            $i++;
            $right = $parseTerm();

            if ($op === '+') {
                $value += $right;
            } else {
                $value -= $right;
            }
        }

        return $value;
    };

    $result = $parseExpression();

    if (!is_finite($result)) {
        return '';
    }

    $result = round($result, 6);
    $text = rtrim(rtrim(number_format($result, 6, '.', ''), '0'), '.');

    return $text === '-0' ? '0' : $text;
};

$columns = array_values(array_map(static function ($column, $index) use ($normalizeAlign, $normalizeType) {
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
        'type' => $normalizeType($column['type'] ?? 'text'),
        'formula' => trim((string)($column['formula'] ?? '')),
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

$calculateFormula = static function (string $formula, array $row) use ($valueToNumber, $evalMathExpression): string {
    $formula = trim($formula);

    if ($formula === '') {
        return '';
    }

    $cells = is_array($row['cells'] ?? null) ? $row['cells'] : [];

    $expression = preg_replace_callback('/\b[A-Za-z_][A-Za-z0-9_]*\b/', static function ($matches) use ($cells, $valueToNumber) {
        $columnId = $matches[0];
        return (string)$valueToNumber($cells[$columnId] ?? '');
    }, $formula);

    return (string)$evalMathExpression((string)$expression);
};

$tableContent = [
    'title' => $title !== '' ? $title : 'Таблица',
    'columns' => $columns,
    'rows' => $normalizedRows,
];

$contentJson = json_encode($tableContent, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
$propsJson = json_encode(is_array($props ?? null) ? $props : [], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

$blockId = (int)($block['id'] ?? 0);

$renderViewCell = static function (array $column, array $row) use ($valueToText, $calculateFormula): string {
    $cells = is_array($row['cells'] ?? null) ? $row['cells'] : [];
    $value = $cells[$column['id']] ?? '';
    $type = (string)($column['type'] ?? 'text');

    if ($type === 'formula') {
        return sb_public_h($calculateFormula((string)($column['formula'] ?? ''), $row));
    }

    if ($type === 'link') {
        $text = is_array($value) ? trim((string)($value['text'] ?? '')) : $valueToText($value);
        $url = is_array($value) ? trim((string)($value['url'] ?? '')) : $valueToText($value);

        if ($text === '') {
            $text = $url;
        }

        if ($url === '') {
            return sb_public_h($text);
        }

        $isSafeUrl = (bool)preg_match('~^(https?://|/|mailto:)~i', $url);

        if (!$isSafeUrl) {
            return sb_public_h($text);
        }

        return '<a href="' . sb_public_h($url) . '" target="_blank" rel="noopener noreferrer">' . sb_public_h($text) . '</a>';
    }

    if ($type === 'image') {
        $src = is_array($value) ? trim((string)($value['src'] ?? '')) : $valueToText($value);
        $alt = is_array($value) ? trim((string)($value['alt'] ?? '')) : '';

        if ($src === '') {
            return '';
        }

        $isSafeSrc = (bool)preg_match('~^(https?://|/)~i', $src);

        if (!$isSafeSrc) {
            return '';
        }

        return '<img class="sb-public-table-image" src="' . sb_public_h($src) . '" alt="' . sb_public_h($alt) . '">';
    }

    return nl2br(sb_public_h($valueToText($value)));
};
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
                            data-column-type-value="<?= sb_public_h($column['type']) ?>"
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
                                    <span class="sb-public-table-column-code"><?= sb_public_h($column['id']) ?></span>

                                    <select class="sb-public-table-type-select" data-column-type>
                                        <option value="text"<?= $column['type'] === 'text' ? ' selected' : '' ?>>Текст</option>
                                        <option value="number"<?= $column['type'] === 'number' ? ' selected' : '' ?>>Число</option>
                                        <option value="date"<?= $column['type'] === 'date' ? ' selected' : '' ?>>Дата</option>
                                        <option value="link"<?= $column['type'] === 'link' ? ' selected' : '' ?>>Гиперссылка</option>
                                        <option value="image"<?= $column['type'] === 'image' ? ' selected' : '' ?>>Рисунок</option>
                                        <option value="formula"<?= $column['type'] === 'formula' ? ' selected' : '' ?>>Формула</option>
                                    </select>

                                    <input
                                        class="sb-public-table-formula-input"
                                        type="text"
                                        value="<?= sb_public_h($column['formula']) ?>"
                                        placeholder="Например: col_1 * col_2"
                                        data-column-formula
                                        style="<?= $column['type'] === 'formula' ? '' : 'display:none;' ?>"
                                    >

                                    <select class="sb-public-table-align-select" data-column-align>
                                        <option value="left"<?= $column['align'] === 'left' ? ' selected' : '' ?>>Слева</option>
                                        <option value="center"<?= $column['align'] === 'center' ? ' selected' : '' ?>>Центр</option>
                                        <option value="right"<?= $column['align'] === 'right' ? ' selected' : '' ?>>Справа</option>
                                    </select>

                                    <button class="sb-public-table-column-delete" type="button" data-table-delete-column title="Удалить столбец">
                                        Удалить столбец
                                    </button>
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
                                <?php
                                $cellValue = $cells[$column['id']] ?? '';
                                $type = $column['type'];
                                ?>
                                <td
                                    data-column-id="<?= sb_public_h($column['id']) ?>"
                                    data-column-type="<?= sb_public_h($type) ?>"
                                    style="text-align:<?= sb_public_h($column['align']) ?>"
                                    <?php if ($isEditMode && !in_array($type, ['link', 'image', 'formula'], true)): ?>
                                        contenteditable="true"
                                        data-cell-editable
                                    <?php endif; ?>
                                >
                                    <?php if (!$isEditMode): ?>
                                        <?= $renderViewCell($column, $row) ?>
                                    <?php elseif ($type === 'link'): ?>
                                        <?php
                                        $linkText = is_array($cellValue) ? trim((string)($cellValue['text'] ?? '')) : $valueToText($cellValue);
                                        $linkUrl = is_array($cellValue) ? trim((string)($cellValue['url'] ?? '')) : $valueToText($cellValue);
                                        ?>
                                        <div class="sb-public-table-cell-link" data-link-cell>
                                            <input type="text" data-link-text value="<?= sb_public_h($linkText) ?>" placeholder="Текст ссылки">
                                            <input type="text" data-link-url value="<?= sb_public_h($linkUrl) ?>" placeholder="https://...">
                                        </div>
                                    <?php elseif ($type === 'image'): ?>
                                        <?php
                                        $imgSrc = is_array($cellValue) ? trim((string)($cellValue['src'] ?? '')) : $valueToText($cellValue);
                                        $imgAlt = is_array($cellValue) ? trim((string)($cellValue['alt'] ?? '')) : '';
                                        ?>
                                        <div class="sb-public-table-cell-image" data-image-cell>
                                            <input type="text" data-image-src value="<?= sb_public_h($imgSrc) ?>" placeholder="/upload/... или https://...">
                                            <input type="text" data-image-alt value="<?= sb_public_h($imgAlt) ?>" placeholder="Описание">
                                        </div>
                                    <?php elseif ($type === 'formula'): ?>
                                        <span class="sb-public-table-formula-value" data-formula-cell>
                                            <?= sb_public_h($calculateFormula($column['formula'], $row)) ?>
                                        </span>
                                    <?php else: ?>
                                        <?= nl2br(sb_public_h($valueToText($cellValue))) ?>
                                    <?php endif; ?>
                                </td>
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
    window.SB_TABLE_EDIT_LOADED = 'v9-column-types';

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

        if (align !== 'left' && align !== 'center' && align !== 'right') {
            return 'left';
        }

        return align;
    }

    function normalizeType(type) {
        type = String(type || 'text');

        if (['text', 'number', 'date', 'link', 'image', 'formula'].indexOf(type) === -1) {
            return 'text';
        }

        return type;
    }

    function textValue(node) {
        return String(node ? (node.innerText || node.textContent || '') : '')
            .replace(/\u00a0/g, ' ')
            .trim();
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

    function getColumnTypeFromTh(th) {
        var select = th.querySelector('[data-column-type]');

        if (select) {
            return normalizeType(select.value);
        }

        return normalizeType(th.getAttribute('data-column-type-value') || 'text');
    }

    function getColumnFormulaFromTh(th) {
        var input = th.querySelector('[data-column-formula]');

        if (input) {
            return String(input.value || '').trim();
        }

        return '';
    }

    function valueToText(value) {
        if (value && typeof value === 'object') {
            if (value.text) return String(value.text);
            if (value.url) return String(value.url);
            if (value.src) return String(value.src);
            if (value.alt) return String(value.alt);
            return '';
        }

        return String(value || '');
    }

    function valueToNumber(value) {
        var text = valueToText(value)
            .replace(/\s+/g, '')
            .replace(',', '.');

        var number = Number(text);

        return Number.isFinite(number) ? number : 0;
    }

    function evaluateMathExpression(expression) {
        expression = String(expression || '').replace(/,/g, '.').trim();

        if (!expression) {
            return '';
        }

        if (!/^[0-9+\-*\/().\s]+$/.test(expression)) {
            return '';
        }

        var tokens = expression.match(/\d+(?:\.\d+)?|[+\-*\/()]/g) || [];

        if (!tokens.length) {
            return '';
        }

        var raw = expression.replace(/\s+/g, '');
        var joined = tokens.join('');

        if (raw !== joined) {
            return '';
        }

        var i = 0;

        function parseFactor() {
            var token = tokens[i];

            if (token === '+') {
                i++;
                return parseFactor();
            }

            if (token === '-') {
                i++;
                return -parseFactor();
            }

            if (token === '(') {
                i++;
                var value = parseExpression();

                if (tokens[i] === ')') {
                    i++;
                }

                return value;
            }

            i++;
            return Number(token || 0);
        }

        function parseTerm() {
            var value = parseFactor();

            while (tokens[i] === '*' || tokens[i] === '/') {
                var op = tokens[i];
                i++;

                var right = parseFactor();

                if (op === '*') {
                    value *= right;
                } else {
                    if (Math.abs(right) < 0.0000001) {
                        return 0;
                    }

                    value /= right;
                }
            }

            return value;
        }

        function parseExpression() {
            var value = parseTerm();

            while (tokens[i] === '+' || tokens[i] === '-') {
                var op = tokens[i];
                i++;

                var right = parseTerm();

                if (op === '+') {
                    value += right;
                } else {
                    value -= right;
                }
            }

            return value;
        }

        var result = parseExpression();

        if (!Number.isFinite(result)) {
            return '';
        }

        result = Math.round(result * 1000000) / 1000000;

        return String(result).replace(/\.0+$/, '');
    }

    function calculateFormula(content, row, formula) {
        formula = String(formula || '').trim();

        if (!formula) {
            return '';
        }

        var cells = row && row.cells ? row.cells : {};

        var expression = formula.replace(/\b[A-Za-z_][A-Za-z0-9_]*\b/g, function (columnId) {
            return String(valueToNumber(cells[columnId]));
        });

        return evaluateMathExpression(expression);
    }

    function getCellValueFromTd(td, column) {
        var type = normalizeType(column.type);

        if (type === 'link') {
            return {
                text: String((td.querySelector('[data-link-text]') || {}).value || '').trim(),
                url: String((td.querySelector('[data-link-url]') || {}).value || '').trim()
            };
        }

        if (type === 'image') {
            return {
                src: String((td.querySelector('[data-image-src]') || {}).value || '').trim(),
                alt: String((td.querySelector('[data-image-alt]') || {}).value || '').trim()
            };
        }

        if (type === 'formula') {
            return '';
        }

        return textValue(td);
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
                th.setAttribute('data-column-id', columnId);
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
                align: getColumnAlignFromTh(th),
                type: getColumnTypeFromTh(th),
                formula: getColumnFormulaFromTh(th)
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

                if (!td) {
                    return;
                }

                if (column.type !== 'formula') {
                    cells[column.id] = getCellValueFromTd(td, column);
                }
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

    function updateFormulaCells(root) {
        var content = collectContentFromDom(root);
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        var formulaColumns = content.columns.filter(function (column) {
            return column.type === 'formula';
        });

        if (!formulaColumns.length) {
            setContent(root, content);
            return;
        }

        table.querySelectorAll('tbody tr[data-row-id]').forEach(function (tr) {
            var rowId = String(tr.getAttribute('data-row-id') || '');
            var row = content.rows.find(function (item) {
                return String(item.id || '') === rowId;
            });

            if (!row) {
                return;
            }

            formulaColumns.forEach(function (column) {
                var td = tr.querySelector('td[data-column-id="' + cssEscape(column.id) + '"]');
                var cell = td ? td.querySelector('[data-formula-cell]') : null;

                if (cell) {
                    cell.textContent = calculateFormula(content, row, column.formula);
                }
            });
        });

        setContent(root, content);
    }

    function createColumnSelect(value) {
        var select = document.createElement('select');
        select.className = 'sb-public-table-type-select';
        select.setAttribute('data-column-type', '');

        select.innerHTML = ''
            + '<option value="text">Текст</option>'
            + '<option value="number">Число</option>'
            + '<option value="date">Дата</option>'
            + '<option value="link">Гиперссылка</option>'
            + '<option value="image">Рисунок</option>'
            + '<option value="formula">Формула</option>';

        select.value = normalizeType(value);

        return select;
    }

    function createAlignSelect(value) {
        var select = document.createElement('select');
        select.className = 'sb-public-table-align-select';
        select.setAttribute('data-column-align', '');

        select.innerHTML = ''
            + '<option value="left">Слева</option>'
            + '<option value="center">Центр</option>'
            + '<option value="right">Справа</option>';

        select.value = normalizeAlign(value);

        return select;
    }

    function renderTableFromContent(root) {
        var content = getContent(root);
        var table = root.querySelector('.sb-public-table');

        if (!table) {
            return;
        }

        var columns = Array.isArray(content.columns) ? content.columns : [];
        var rows = Array.isArray(content.rows) ? content.rows : [];
        var hasControlCol = !!root.querySelector('.sb-public-table__control-col') || root.querySelector('.sb-public-table--editable');

        var colgroup = table.querySelector('colgroup');

        if (!colgroup) {
            colgroup = document.createElement('colgroup');
            table.insertBefore(colgroup, table.firstChild);
        }

        colgroup.innerHTML = '';

        if (hasControlCol) {
            var controlCol = document.createElement('col');
            controlCol.className = 'sb-public-table__control-col';
            controlCol.style.width = '72px';
            colgroup.appendChild(controlCol);
        }

        columns.forEach(function (column) {
            var col = document.createElement('col');
            col.setAttribute('data-column-id', column.id);
            col.setAttribute('width', String(clampWidth(column.width || 160)));
            col.style.width = clampWidth(column.width || 160) + 'px';
            colgroup.appendChild(col);
        });

        var thead = table.querySelector('thead');

        if (!thead) {
            thead = document.createElement('thead');
            table.appendChild(thead);
        }

        var headRow = thead.querySelector('tr');

        if (!headRow) {
            headRow = document.createElement('tr');
            thead.appendChild(headRow);
        }

        headRow.innerHTML = '';

        if (hasControlCol) {
            var controlTh = document.createElement('th');
            controlTh.className = 'sb-public-table__control-th';
            controlTh.textContent = '№';
            headRow.appendChild(controlTh);
        }

        columns.forEach(function (column) {
            column.type = normalizeType(column.type);
            column.align = normalizeAlign(column.align);

            var th = document.createElement('th');

            th.setAttribute('data-column-id', column.id);
            th.setAttribute('data-column-align-value', column.align);
            th.setAttribute('data-column-type-value', column.type);
            th.style.textAlign = column.align;

            var inner = document.createElement('div');
            inner.className = 'sb-public-table-th-inner';

            var label = document.createElement('span');
            label.className = 'sb-public-table__th-text';
            label.setAttribute('contenteditable', 'true');
            label.setAttribute('data-column-label', '');
            label.textContent = column.label || 'Столбец';

            var code = document.createElement('span');
            code.className = 'sb-public-table-column-code';
            code.textContent = column.id;

            var typeSelect = createColumnSelect(column.type);

            var formulaInput = document.createElement('input');
            formulaInput.className = 'sb-public-table-formula-input';
            formulaInput.type = 'text';
            formulaInput.placeholder = 'Например: col_1 * col_2';
            formulaInput.value = column.formula || '';
            formulaInput.setAttribute('data-column-formula', '');

            if (column.type !== 'formula') {
                formulaInput.style.display = 'none';
            }

            var alignSelect = createAlignSelect(column.align);

            var deleteButton = document.createElement('button');
            deleteButton.className = 'sb-public-table-column-delete';
            deleteButton.type = 'button';
            deleteButton.setAttribute('data-table-delete-column', '');
            deleteButton.title = 'Удалить столбец';
            deleteButton.textContent = 'Удалить столбец';

            inner.appendChild(label);
            inner.appendChild(code);
            inner.appendChild(typeSelect);
            inner.appendChild(formulaInput);
            inner.appendChild(alignSelect);
            inner.appendChild(deleteButton);

            var resizer = document.createElement('span');
            resizer.className = 'sb-public-table-resizer';
            resizer.setAttribute('data-column-resizer', '');

            th.appendChild(inner);
            th.appendChild(resizer);

            headRow.appendChild(th);
        });

        var tbody = table.querySelector('tbody');

        if (!tbody) {
            tbody = document.createElement('tbody');
            table.appendChild(tbody);
        }

        tbody.innerHTML = '';

        if (!rows.length) {
            var emptyTr = document.createElement('tr');
            emptyTr.setAttribute('data-empty-row', '');

            var emptyTd = document.createElement('td');
            emptyTd.setAttribute('colspan', String(columns.length + (hasControlCol ? 1 : 0)));
            emptyTd.textContent = 'Нет данных';

            emptyTr.appendChild(emptyTd);
            tbody.appendChild(emptyTr);
        } else {
            rows.forEach(function (row, rowIndex) {
                var tr = document.createElement('tr');
                tr.setAttribute('data-row-id', row.id || ('row_' + (rowIndex + 1)));

                if (hasControlCol) {
                    var controlTd = document.createElement('td');
                    controlTd.className = 'sb-public-table__control-td';
                    controlTd.innerHTML = ''
                        + '<div class="sb-public-table-row-actions">'
                        + '  <span class="sb-public-table-row-num">' + (rowIndex + 1) + '</span>'
                        + '  <button type="button" class="sb-public-table-row-delete" data-table-delete-row title="Удалить строку">×</button>'
                        + '</div>';

                    tr.appendChild(controlTd);
                }

                columns.forEach(function (column) {
                    var td = document.createElement('td');
                    var type = normalizeType(column.type);
                    var value = row.cells ? row.cells[column.id] : '';

                    td.setAttribute('data-column-id', column.id);
                    td.setAttribute('data-column-type', type);
                    td.style.textAlign = normalizeAlign(column.align);

                    if (type === 'link') {
                        var linkWrap = document.createElement('div');
                        linkWrap.className = 'sb-public-table-cell-link';
                        linkWrap.setAttribute('data-link-cell', '');

                        var linkText = document.createElement('input');
                        linkText.type = 'text';
                        linkText.setAttribute('data-link-text', '');
                        linkText.placeholder = 'Текст ссылки';
                        linkText.value = value && typeof value === 'object' ? String(value.text || '') : valueToText(value);

                        var linkUrl = document.createElement('input');
                        linkUrl.type = 'text';
                        linkUrl.setAttribute('data-link-url', '');
                        linkUrl.placeholder = 'https://...';
                        linkUrl.value = value && typeof value === 'object' ? String(value.url || '') : valueToText(value);

                        linkWrap.appendChild(linkText);
                        linkWrap.appendChild(linkUrl);
                        td.appendChild(linkWrap);
                    } else if (type === 'image') {
                        var imageWrap = document.createElement('div');
                        imageWrap.className = 'sb-public-table-cell-image';
                        imageWrap.setAttribute('data-image-cell', '');

                        var imageSrc = document.createElement('input');
                        imageSrc.type = 'text';
                        imageSrc.setAttribute('data-image-src', '');
                        imageSrc.placeholder = '/upload/... или https://...';
                        imageSrc.value = value && typeof value === 'object' ? String(value.src || '') : valueToText(value);

                        var imageAlt = document.createElement('input');
                        imageAlt.type = 'text';
                        imageAlt.setAttribute('data-image-alt', '');
                        imageAlt.placeholder = 'Описание';
                        imageAlt.value = value && typeof value === 'object' ? String(value.alt || '') : '';

                        imageWrap.appendChild(imageSrc);
                        imageWrap.appendChild(imageAlt);
                        td.appendChild(imageWrap);
                    } else if (type === 'formula') {
                        var formulaSpan = document.createElement('span');
                        formulaSpan.className = 'sb-public-table-formula-value';
                        formulaSpan.setAttribute('data-formula-cell', '');
                        formulaSpan.textContent = calculateFormula(content, row, column.formula || '');
                        td.appendChild(formulaSpan);
                    } else {
                        td.setAttribute('contenteditable', 'true');
                        td.setAttribute('data-cell-editable', '');
                        td.textContent = valueToText(value);
                    }

                    tr.appendChild(td);
                });

                tbody.appendChild(tr);
            });
        }

        setContent(root, collectContentFromDom(root));
        applyWidths(root);
        applyAllAligns(root);
        renumberRows(root);
        updateFormulaCells(root);
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
        var content = collectContentFromDom(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];
        var rows = Array.isArray(content.rows) ? content.rows : [];

        var newIndex = columns.length + 1;
        var newColumnId = 'col_' + Date.now();

        columns.push({
            id: newColumnId,
            label: 'Столбец ' + newIndex,
            width: 160,
            align: 'left',
            type: 'text',
            formula: ''
        });

        rows = rows.map(function (row) {
            row.cells = row.cells || {};
            row.cells[newColumnId] = '';
            return row;
        });

        content.columns = columns;
        content.rows = rows;

        setContent(root, content);
        renderTableFromContent(root);
        setDirty(root, true);
    }

    function deleteColumn(root, columnId) {
        var content = collectContentFromDom(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];
        var rows = Array.isArray(content.rows) ? content.rows : [];

        if (columns.length <= 1) {
            alert('Нельзя удалить последний столбец');
            return;
        }

        var column = columns.find(function (item) {
            return String(item.id || '') === String(columnId);
        });

        var columnName = column && column.label ? column.label : columnId;

        if (!confirm('Удалить столбец "' + columnName + '"? Данные в этом столбце будут удалены.')) {
            return;
        }

        columns = columns.filter(function (item) {
            return String(item.id || '') !== String(columnId);
        });

        rows = rows.map(function (row) {
            row.cells = row.cells || {};
            delete row.cells[columnId];
            return row;
        });

        content.columns = columns;
        content.rows = rows;

        setContent(root, content);
        renderTableFromContent(root);
        setDirty(root, true);
    }

    function addRow(root) {
        var content = collectContentFromDom(root);
        var columns = Array.isArray(content.columns) ? content.columns : [];
        var rows = Array.isArray(content.rows) ? content.rows : [];

        var rowId = 'row_' + Date.now();
        var cells = {};

        columns.forEach(function (column) {
            if (column.type !== 'formula') {
                cells[column.id] = '';
            }
        });

        rows.push({
            id: rowId,
            cells: cells
        });

        content.rows = rows;

        setContent(root, content);
        renderTableFromContent(root);
        setDirty(root, true);
    }

    function deleteRow(root, btn) {
        var tr = btn.closest('tr[data-row-id]');

        if (!tr) {
            return;
        }

        var rowId = String(tr.getAttribute('data-row-id') || '');

        if (!confirm('Удалить строку?')) {
            return;
        }

        var content = collectContentFromDom(root);
        var rows = Array.isArray(content.rows) ? content.rows : [];

        content.rows = rows.filter(function (row) {
            return String(row.id || '') !== rowId;
        });

        setContent(root, content);
        renderTableFromContent(root);
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
        var content = collectContentFromDom(root);
        setContent(root, content);

        applyWidths(root);
        applyAllAligns(root);
        renumberRows(root);
        updateFormulaCells(root);
    }

    function initAllTables() {
        document.querySelectorAll('[data-public-editable-table]').forEach(initTable);
    }

    document.addEventListener('input', function (e) {
        var root = e.target.closest('[data-public-editable-table]');

        if (!root) {
            return;
        }

        if (
            e.target.matches('[data-table-title-input]') ||
            e.target.matches('[data-column-label]') ||
            e.target.matches('[data-cell-editable]') ||
            e.target.matches('[data-column-formula]') ||
            e.target.matches('[data-link-text]') ||
            e.target.matches('[data-link-url]') ||
            e.target.matches('[data-image-src]') ||
            e.target.matches('[data-image-alt]')
        ) {
            var content = collectContentFromDom(root);
            setContent(root, content);
            updateFormulaCells(root);
            setDirty(root, true);
        }
    }, true);

    document.addEventListener('change', function (e) {
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

        var select = e.target.closest('[data-column-align]');

        if (!select) {
            return;
        }

        var root = select.closest('[data-public-editable-table]');
        var th = select.closest('th[data-column-id]');
        var columnId = th ? String(th.getAttribute('data-column-id') || '') : '';

        if (!root || !columnId) {
            return;
        }

        e.stopImmediatePropagation();

        applyColumnAlign(root, columnId, normalizeAlign(select.value));

        setContent(root, collectContentFromDom(root));
        setDirty(root, true);
    }, true);

    document.addEventListener('click', function (e) {
        var deleteColumnBtn = e.target.closest('[data-table-delete-column]');

        if (deleteColumnBtn) {
            var deleteRoot = deleteColumnBtn.closest('[data-public-editable-table]');
            var deleteTh = deleteColumnBtn.closest('th[data-column-id]');
            var deleteColumnId = deleteTh ? String(deleteTh.getAttribute('data-column-id') || '') : '';

            if (deleteRoot && deleteColumnId) {
                e.preventDefault();
                e.stopImmediatePropagation();
                deleteColumn(deleteRoot, deleteColumnId);
            }

            return;
        }

        var root = e.target.closest('[data-public-editable-table]');

        if (!root) {
            return;
        }

        if (e.target.closest('[data-table-add-column]')) {
            e.preventDefault();
            e.stopImmediatePropagation();
            addColumn(root);
            return;
        }

        if (e.target.closest('[data-table-add-row]')) {
            e.preventDefault();
            e.stopImmediatePropagation();
            addRow(root);
            return;
        }

        if (e.target.closest('[data-table-save-all]')) {
            e.preventDefault();
            e.stopImmediatePropagation();
            saveBlock(root);
            return;
        }

        var deleteBtn = e.target.closest('[data-table-delete-row]');

        if (deleteBtn) {
            e.preventDefault();
            e.stopImmediatePropagation();
            deleteRow(root, deleteBtn);
        }
    }, true);

    document.addEventListener('keydown', function (e) {
        if (e.target.matches('[data-cell-editable], [data-column-label]')) {
            if (e.key === 'Enter' && !e.shiftKey) {
                e.preventDefault();
                e.target.blur();
            }
        }
    }, true);

    document.addEventListener('mousedown', function (e) {
        if (
            e.target.closest('[data-column-align]') ||
            e.target.closest('[data-column-type]') ||
            e.target.closest('[data-column-formula]') ||
            e.target.closest('[data-table-delete-column]') ||
            e.target.closest('[data-link-text]') ||
            e.target.closest('[data-link-url]') ||
            e.target.closest('[data-image-src]') ||
            e.target.closest('[data-image-alt]')
        ) {
            return;
        }

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

3. Добавь CSS

В конец файла:

/local/sitebuilder/assets/public/public.css

добавь:

/* =========================================================
   Public table data types
   ========================================================= */

.sb-public-table-column-code {
    display: inline-flex;
    width: fit-content;
    max-width: 100%;
    padding: 2px 6px;
    border-radius: 999px;
    background: #e2e8f0;
    color: #475569;
    font-size: 10px;
    font-weight: 900;
    line-height: 1.3;
}

.sb-public-table-type-select,
.sb-public-table-formula-input {
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

.sb-public-table-formula-input {
    font-family: monospace;
    font-weight: 700;
}

.sb-public-table-cell-link,
.sb-public-table-cell-image {
    display: grid;
    gap: 5px;
}

.sb-public-table-cell-link input,
.sb-public-table-cell-image input {
    width: 100%;
    min-height: 30px;
    box-sizing: border-box;
    padding: 5px 8px;
    border: 1px solid rgba(148, 163, 184, .7);
    border-radius: 8px;
    background: #fff;
    color: #0f172a;
    font-size: 12px;
    font-weight: 700;
}

.sb-public-table-formula-value {
    display: inline-flex;
    min-height: 20px;
    align-items: center;
    font-weight: 900;
    color: #0f172a;
}

.sb-public-table-image {
    display: block;
    max-width: 160px;
    max-height: 120px;
    border-radius: 10px;
    object-fit: contain;
}

После замены открой public с edit=Y, сделай Ctrl + F5.

Проверка:

window.SB_TABLE_EDIT_LOADED

Должно быть:

"v9-column-types"

Следующим этапом после проверки добавим лимит строк + пагинацию.