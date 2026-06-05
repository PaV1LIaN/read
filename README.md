Понял. Тут нужно понять, где именно не отображается:

1. В редакторе нет кнопки “Таблица” — проблема в editor.php.


2. Кнопка есть, но блок не создаётся / не появляется — проблема в 30-blocks.js или кэше.


3. В редакторе блок есть, но на публичной странице таблицы нет — проблема в table.php или public_render.php.



Сейчас сделай по порядку.

1. Обнови версии JS в editor.php

Внизу editor.php поставь такие версии:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/00-core.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/10-sections.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/20-pages.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/30-blocks.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/40-access.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/50-template.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/60-events.js?v=3"></script>

И для CSS:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.css?v=3">

Потом в редакторе сделай Ctrl + F5.


---

2. Проверь, создан ли файл публичного шаблона

Должен быть файл:

/local/sitebuilder/views/blocks/table.php

Содержимое сделай таким:

<?php
$title = trim((string)($content['title'] ?? ''));

$columns = is_array($content['columns'] ?? null) ? $content['columns'] : [];
$rows = is_array($content['rows'] ?? null) ? $content['rows'] : [];

if (empty($columns)) {
    return;
}

$columns = array_values(array_map(static function ($column, $index) {
    $id = trim((string)($column['id'] ?? ''));

    if ($id === '') {
        $id = 'col_' . ($index + 1);
    }

    return [
        'id' => $id,
        'label' => trim((string)($column['label'] ?? ('Столбец ' . ($index + 1)))),
    ];
}, $columns, array_keys($columns)));
?>

<section class="sb-block sb-block--table">
    <?php if ($title !== ''): ?>
        <h2 class="sb-public-table__title"><?= sb_public_h($title) ?></h2>
    <?php endif; ?>

    <div class="sb-public-table-wrap">
        <table class="sb-public-table">
            <thead>
                <tr>
                    <?php foreach ($columns as $column): ?>
                        <th><?= sb_public_h($column['label']) ?></th>
                    <?php endforeach; ?>
                </tr>
            </thead>

            <tbody>
                <?php if (!empty($rows)): ?>
                    <?php foreach ($rows as $row): ?>
                        <?php $cells = is_array($row['cells'] ?? null) ? $row['cells'] : []; ?>
                        <tr>
                            <?php foreach ($columns as $column): ?>
                                <td><?= nl2br(sb_public_h((string)($cells[$column['id']] ?? ''))) ?></td>
                            <?php endforeach; ?>
                        </tr>
                    <?php endforeach; ?>
                <?php else: ?>
                    <tr>
                        <td colspan="<?= count($columns) ?>">Нет данных</td>
                    </tr>
                <?php endif; ?>
            </tbody>
        </table>
    </div>
</section>


---

3. Проверь, что блок реально создался

После нажатия Таблица в редакторе справа/в ответе API должен появиться блок с типом:

"type": "table"

Если блока с type: "table" нет, значит не сработал JS.

Открой консоль браузера на странице редактора и введи:

typeof normalizeTableContent

Должно вернуть:

"function"

Если вернуло:

"undefined"

значит браузер не подхватил новый 30-blocks.js. Тогда проверь, что в editor.php подключается именно:

/assets/admin/editor/30-blocks.js?v=3

и сделай Ctrl + F5.


---

4. Если в редакторе таблица есть, но на публичной странице нет

На публичной странице нажми Ctrl + U и найди:

sb-block--table

Если не находится, значит публичка не получила блок table.

Тогда в public_render.php нужно убедиться, что content нормально декодируется. В функции:

function sb_public_render_block(array $block, array $context = []): string

должно быть так:

$block = sb_normalize_block_record($block);
$content = sb_public_to_array($block['content'] ?? []);
$props = sb_public_to_array($block['props'] ?? []);

А не так:

$content = (array)($block['content'] ?? []);
$props = (array)($block['props'] ?? []);

Если у тебя старый вариант — замени.


---

5. Добавь стили, если таблица есть, но выглядит пустой/кривой

В конец:

/local/sitebuilder/assets/public/public.css

добавь:

.sb-block--table {
    width: 100%;
    min-width: 0;
}

.sb-public-table__title {
    margin: 0 0 14px;
    color: #111827;
    font-size: 24px;
    line-height: 1.25;
    font-weight: 900;
}

.sb-public-table-wrap {
    width: 100%;
    min-width: 0;
    overflow-x: auto;
    border: 1px solid rgba(148, 163, 184, .45);
    border-radius: 16px;
    background: #fff;
}

.sb-public-table {
    width: 100%;
    min-width: 640px;
    border-collapse: collapse;
    font-size: 14px;
}

.sb-public-table th {
    padding: 14px 16px;
    background: #f8fafc;
    color: #334155;
    font-weight: 900;
    text-align: left;
    border-bottom: 1px solid rgba(148, 163, 184, .45);
    white-space: nowrap;
}

.sb-public-table td {
    padding: 13px 16px;
    color: #334155;
    border-bottom: 1px solid rgba(226, 232, 240, .9);
    vertical-align: top;
}

.sb-public-table tbody tr:last-child td {
    border-bottom: 0;
}

.sb-public-table tbody tr:hover td {
    background: #f8fafc;
}

И в public_page.php обнови версию:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=12">


---

Самая частая причина сейчас — кэш: 30-blocks.js подключён старой версией, поэтому редактор ещё не знает тип table. Проверка typeof normalizeTableContent сразу покажет это.