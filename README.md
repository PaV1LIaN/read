Да, дату делаем именно в формате:

dd.mm.yyyy

То есть:

11.06.2026

А не 2026-06-11.

Начинаем с первого файла — table.php. После него нужно будет заменить table-edit.js, потому что логика типов должна совпадать и на PHP, и на JS.


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

$normalizeDate = static function ($value) use ($valueToText): string {
    $value = $valueToText($value);

    if ($value === '') {
        return '';
    }

    $value = trim($value);

    if (preg_match('/^(\d{1,2})\.(\d{1,2})\.(\d{4})$/', $value, $m)) {
        $day = (int)$m[1];
        $month = (int)$m[2];
        $year = (int)$m[3];

        if (checkdate($month, $day, $year)) {
            return sprintf('%02d.%02d.%04d', $day, $month, $year);
        }

        return '';
    }

    if (preg_match('/^(\d{4})-(\d{1,2})-(\d{1,2})$/', $value, $m)) {
        $year = (int)$m[1];
        $month = (int)$m[2];
        $day = (int)$m[3];

        if (checkdate($month, $day, $year)) {
            return sprintf('%02d.%02d.%04d', $day, $month, $year);
        }

        return '';
    }

    return '';
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
        return 'Ошибка';
    }

    preg_match_all('/\d+(?:\.\d+)?|[+\-*\/()]/', $expression, $matches);

    $tokens = $matches[0] ?? [];

    if (empty($tokens)) {
        return '';
    }

    $raw = preg_replace('/\s+/', '', $expression);
    $joined = implode('', $tokens);

    if ($raw !== $joined) {
        return 'Ошибка';
    }

    $i = 0;
    $count = count($tokens);

    $parseExpression = null;
    $parseTerm = null;
    $parseFactor = null;
    $hasError = false;

    $parseFactor = static function () use (&$tokens, &$i, &$count, &$parseExpression, &$parseFactor, &$hasError) {
        if ($i >= $count) {
            $hasError = true;
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
            } else {
                $hasError = true;
            }

            return $value;
        }

        if (!is_numeric($token)) {
            $hasError = true;
            return 0.0;
        }

        $i++;

        return (float)$token;
    };

    $parseTerm = static function () use (&$tokens, &$i, &$count, &$parseFactor, &$hasError) {
        $value = $parseFactor();

        while ($i < $count && ($tokens[$i] === '*' || $tokens[$i] === '/')) {
            $op = $tokens[$i];
            $i++;
            $right = $parseFactor();

            if ($op === '*') {
                $value *= $right;
            } else {
                if (abs($right) < 0.0000001) {
                    $hasError = true;
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

    if ($hasError || $i < $count || !is_finite($result)) {
        return 'Ошибка';
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

$renderViewCell = static function (array $column, array $row) use ($valueToText, $normalizeDate, $calculateFormula): string {
    $cells = is_array($row['cells'] ?? null) ? $row['cells'] : [];
    $value = $cells[$column['id']] ?? '';
    $type = (string)($column['type'] ?? 'text');

    if ($type === 'formula') {
        return sb_public_h($calculateFormula((string)($column['formula'] ?? ''), $row));
    }

    if ($type === 'date') {
        return sb_public_h($normalizeDate($value));
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
                                    <?php if ($isEditMode && $type === 'text'): ?>
                                        contenteditable="true"
                                        data-cell-editable
                                    <?php endif; ?>
                                >
                                    <?php if (!$isEditMode): ?>
                                        <?= $renderViewCell($column, $row) ?>

                                    <?php elseif ($type === 'number'): ?>
                                        <?php
                                        $numberValue = str_replace([' ', ','], ['', '.'], $valueToText($cellValue));

                                        if (!is_numeric($numberValue)) {
                                            $numberValue = '';
                                        }
                                        ?>

                                        <input
                                            class="sb-public-table-cell-input"
                                            type="text"
                                            inputmode="decimal"
                                            value="<?= sb_public_h($numberValue) ?>"
                                            placeholder="0"
                                            data-number-cell
                                        >

                                    <?php elseif ($type === 'date'): ?>
                                        <input
                                            class="sb-public-table-cell-input"
                                            type="text"
                                            value="<?= sb_public_h($normalizeDate($cellValue)) ?>"
                                            placeholder="дд.мм.гггг"
                                            maxlength="10"
                                            data-date-cell
                                        >

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

Пока не проверяй окончательно, потому что для этой версии нужен новый table-edit.js. Следующим сообщением пришлю его полностью, уже с проверкой number, date dd.mm.yyyy, link, image и formula.