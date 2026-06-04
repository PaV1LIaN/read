Проблема почти точно в том, что sectionId и column не сохраняются в блоке.
JS выбирает нужную секцию/колонку, но после сохранения/перезагрузки блок снова приходит без этих полей, и рендер кладёт его в первую секцию/первую колонку.

Делаем надёжно: будем хранить привязку и в верхнем уровне блока, и внутри props, чтобы она не терялась при нормализации.


---

1. Исправь PageSectionRepository.php

Файл:

/local/sitebuilder/lib/PageSectionRepository.php

Найди метод:

public static function assignBlock(int $blockId, int $sectionId, int $column, int $userId): array

Замени его полностью на:

public static function assignBlock(int $blockId, int $sectionId, int $column, int $userId): array
{
    $section = self::getById($sectionId);

    if (!$section) {
        throw new RuntimeException('PAGE_SECTION_NOT_FOUND');
    }

    $column = max(1, min(4, $column));

    $blocks = sb_read_blocks();
    $updated = null;

    foreach ($blocks as &$block) {
        if ((int)($block['id'] ?? 0) !== $blockId) {
            continue;
        }

        if ((int)($block['pageId'] ?? 0) !== (int)$section['pageId']) {
            throw new RuntimeException('BLOCK_AND_SECTION_PAGE_MISMATCH');
        }

        $props = is_array($block['props'] ?? null) ? $block['props'] : [];

        $props['sectionId'] = $sectionId;
        $props['column'] = $column;
        $props['_placement'] = [
            'sectionId' => $sectionId,
            'column' => $column,
        ];

        $block['sectionId'] = $sectionId;
        $block['column'] = $column;
        $block['props'] = $props;
        $block['updatedBy'] = $userId;
        $block['updatedAt'] = date('c');

        $updated = $block;
        break;
    }
    unset($block);

    if (!$updated) {
        throw new RuntimeException('BLOCK_NOT_FOUND');
    }

    sb_write_blocks($blocks);

    return $updated;
}


---

2. В этом же файле замени moveBlocksFromSection

Найди:

protected static function moveBlocksFromSection(int $fromSectionId, int $toSectionId, int $userId): void

Замени полностью на:

protected static function moveBlocksFromSection(int $fromSectionId, int $toSectionId, int $userId): void
{
    $blocks = sb_read_blocks();

    foreach ($blocks as &$block) {
        $props = is_array($block['props'] ?? null) ? $block['props'] : [];

        $blockSectionId = (int)($block['sectionId'] ?? 0);

        if ($blockSectionId <= 0) {
            $blockSectionId = (int)($props['sectionId'] ?? 0);
        }

        if ($blockSectionId <= 0 && is_array($props['_placement'] ?? null)) {
            $blockSectionId = (int)($props['_placement']['sectionId'] ?? 0);
        }

        if ($blockSectionId !== $fromSectionId) {
            continue;
        }

        $props['sectionId'] = $toSectionId;
        $props['column'] = 1;
        $props['_placement'] = [
            'sectionId' => $toSectionId,
            'column' => 1,
        ];

        $block['sectionId'] = $toSectionId;
        $block['column'] = 1;
        $block['props'] = $props;
        $block['updatedBy'] = $userId;
        $block['updatedAt'] = date('c');
    }
    unset($block);

    sb_write_blocks($blocks);
}


---

3. В этом же файле замени migratePageBlocksToSection

Найди:

protected static function migratePageBlocksToSection(int $pageId, int $sectionId): void

Замени полностью на:

protected static function migratePageBlocksToSection(int $pageId, int $sectionId): void
{
    if (!function_exists('sb_read_blocks') || !function_exists('sb_write_blocks')) {
        return;
    }

    $blocks = sb_read_blocks();
    $changed = false;

    foreach ($blocks as &$block) {
        if ((int)($block['pageId'] ?? 0) !== $pageId) {
            continue;
        }

        $props = is_array($block['props'] ?? null) ? $block['props'] : [];

        $currentSectionId = (int)($block['sectionId'] ?? 0);

        if ($currentSectionId <= 0) {
            $currentSectionId = (int)($props['sectionId'] ?? 0);
        }

        if ($currentSectionId <= 0 && is_array($props['_placement'] ?? null)) {
            $currentSectionId = (int)($props['_placement']['sectionId'] ?? 0);
        }

        if ($currentSectionId > 0) {
            continue;
        }

        $props['sectionId'] = $sectionId;
        $props['column'] = 1;
        $props['_placement'] = [
            'sectionId' => $sectionId,
            'column' => 1,
        ];

        $block['sectionId'] = $sectionId;
        $block['column'] = 1;
        $block['props'] = $props;

        $changed = true;
    }
    unset($block);

    if ($changed) {
        sb_write_blocks($blocks);
    }
}


---

4. Исправь чтение секции/колонки в editor.js

Файл:

/local/sitebuilder/assets/admin/editor.js

После функции:

function getCurrentBlock() {

вставь:

function getBlockSectionId(block) {
    block = block || {};

    var props = block.props || {};
    var placement = props._placement || {};

    return Number(
        block.sectionId ||
        props.sectionId ||
        placement.sectionId ||
        0
    );
}

function getBlockColumn(block) {
    block = block || {};

    var props = block.props || {};
    var placement = props._placement || {};

    return Number(
        block.column ||
        props.column ||
        placement.column ||
        1
    );
}


---

5. В editor.js замени чтение block.sectionId и block.column

Найди в groupBlocksBySectionAndColumn():

var sectionId = Number(block.sectionId || 0);

Замени на:

var sectionId = getBlockSectionId(block);

Ниже найди:

var column = Number(block.column || 1);

Замени на:

var column = getBlockColumn(block);


---

В fillBlockPlacementForm(block) найди:

var currentSectionId = Number(block.sectionId || 0);

Замени на:

var currentSectionId = getBlockSectionId(block);

Ниже найди:

var currentColumn = Number(block.column || 1);

Замени на:

var currentColumn = getBlockColumn(block);


---

В обработчике выбора блока найди:

var selectedSectionId = Number(selectedBlock.sectionId || 0);
var selectedColumn = Number(selectedBlock.column || 1);

Замени на:

var selectedSectionId = getBlockSectionId(selectedBlock);
var selectedColumn = getBlockColumn(selectedBlock);


---

6. Исправь публичный вывод

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди функцию:

function sb_public_group_blocks_by_section(array $pageBlocks, array $sections): array

Внутри неё найди:

$sectionId = (int)($block['sectionId'] ?? 0);

Замени на:

$props = is_array($block['props'] ?? null) ? $block['props'] : [];
$placement = is_array($props['_placement'] ?? null) ? $props['_placement'] : [];

$sectionId = (int)($block['sectionId'] ?? 0);

if ($sectionId <= 0) {
    $sectionId = (int)($props['sectionId'] ?? 0);
}

if ($sectionId <= 0) {
    $sectionId = (int)($placement['sectionId'] ?? 0);
}


---

Найди функцию:

function sb_public_group_blocks_by_column(array $blocks, int $columns): array

Внутри неё найди:

$column = (int)($block['column'] ?? 1);

Замени на:

$props = is_array($block['props'] ?? null) ? $block['props'] : [];
$placement = is_array($props['_placement'] ?? null) ? $props['_placement'] : [];

$column = (int)($block['column'] ?? 0);

if ($column <= 0) {
    $column = (int)($props['column'] ?? 0);
}

if ($column <= 0) {
    $column = (int)($placement['column'] ?? 0);
}

if ($column <= 0) {
    $column = 1;
}


---

7. Обнови версию JS

В editor.php:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.js?v=5"></script>

И сделай Ctrl + F5.


---

Что проверить

1. Нажми во второй колонке “Добавлять сюда”.


2. Добавь блок “Текст”.


3. Он должен появиться во второй колонке.


4. Обнови страницу редактора.


5. Блок должен остаться во второй колонке.


6. Открой публичную страницу — блок тоже должен быть во второй колонке.



Если после этого всё равно падает в первую колонку, значит sb_write_blocks() полностью выкидывает изменённые props. Тогда надо будет править sb_normalize_block_record() в helpers.php.