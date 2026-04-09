Отлично. Тогда даю рабочий block-пакет.

Сейчас сделаем:

block.list

block.create

block.update

block.delete

block.duplicate

block.move

block.reorder


Это уже даст основу для редактора страниц.


---

1. Обнови /local/sitebuilder/lib/helpers.php

Добавь в конец файла эти функции:

<?php

if (!function_exists('sb_blocks_for_page')) {
    function sb_blocks_for_page(int $pageId): array
    {
        $blocks = array_values(array_filter(sb_read_blocks(), static function ($b) use ($pageId) {
            return (int)($b['pageId'] ?? 0) === $pageId;
        }));

        usort($blocks, static function ($a, $b) {
            $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
            if ($sortCmp !== 0) {
                return $sortCmp;
            }
            return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
        });

        return $blocks;
    }
}

if (!function_exists('sb_next_block_id')) {
    function sb_next_block_id(array $blocks = null): int
    {
        if ($blocks === null) {
            $blocks = sb_read_blocks();
        }

        $maxId = 0;
        foreach ($blocks as $b) {
            $maxId = max($maxId, (int)($b['id'] ?? 0));
        }

        return $maxId + 1;
    }
}

if (!function_exists('sb_next_block_sort')) {
    function sb_next_block_sort(int $pageId, array $blocks = null): int
    {
        if ($blocks === null) {
            $blocks = sb_read_blocks();
        }

        $maxSort = 0;
        foreach ($blocks as $b) {
            if ((int)($b['pageId'] ?? 0) === $pageId) {
                $maxSort = max($maxSort, (int)($b['sort'] ?? 0));
            }
        }

        return $maxSort + 10;
    }
}

if (!function_exists('sb_default_block_content')) {
    function sb_default_block_content(string $type): array
    {
        switch ($type) {
            case 'text':
                return ['html' => '<p>Новый текст</p>'];

            case 'heading':
                return [
                    'text' => 'Новый заголовок',
                    'level' => 'h2',
                    'align' => 'left',
                ];

            case 'image':
                return [
                    'fileId' => 0,
                    'src' => '',
                    'alt' => '',
                    'title' => '',
                    'width' => '',
                    'height' => '',
                    'link' => '',
                ];

            case 'button':
                return [
                    'text' => 'Кнопка',
                    'href' => '#',
                    'target' => '_self',
                    'style' => 'primary',
                    'align' => 'left',
                ];

            case 'spacer':
                return [
                    'height' => 30,
                ];

            case 'columns2':
                return [
                    'leftHtml' => '<p>Левая колонка</p>',
                    'rightHtml' => '<p>Правая колонка</p>',
                    'ratio' => '1:1',
                    'gap' => 24,
                ];

            case 'gallery':
                return [
                    'items' => [],
                    'columns' => 3,
                    'gap' => 16,
                ];

            case 'card':
                return [
                    'title' => 'Карточка',
                    'text' => 'Описание карточки',
                    'imageFileId' => 0,
                    'imageSrc' => '',
                    'buttonText' => '',
                    'buttonHref' => '',
                ];

            case 'cards':
                return [
                    'items' => [],
                    'columns' => 3,
                    'gap' => 24,
                ];

            case 'html':
                return [
                    'html' => '<div>HTML блок</div>',
                ];

            default:
                return [];
        }
    }
}

if (!function_exists('sb_normalize_block_record')) {
    function sb_normalize_block_record(array $block): array
    {
        if (!isset($block['content']) || !is_array($block['content'])) {
            $block['content'] = [];
        }

        if (!isset($block['props']) || !is_array($block['props'])) {
            $block['props'] = [];
        }

        if (!isset($block['type'])) {
            $block['type'] = 'text';
        }

        if (!isset($block['sort'])) {
            $block['sort'] = 500;
        }

        return $block;
    }
}


---

2. Полностью замени /local/sitebuilder/api/handlers/block.php

<?php

global $USER;

if ($action === 'block.list') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_viewer($siteId);

    $blocks = sb_blocks_for_page($pageId);
    $blocks = array_map('sb_normalize_block_record', $blocks);

    sb_json_ok(['blocks' => $blocks]);
}

if ($action === 'block.create') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    $type = trim((string)($_POST['type'] ?? 'text'));

    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }
    if ($type === '') {
        sb_json_error('TYPE_REQUIRED', 422);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();

    $block = [
        'id' => sb_next_block_id($blocks),
        'pageId' => $pageId,
        'type' => $type,
        'sort' => sb_next_block_sort($pageId, $blocks),
        'content' => sb_default_block_content($type),
        'props' => [],
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
        'updatedBy' => (int)$USER->GetID(),
    ];

    $blocks[] = $block;
    sb_write_blocks($blocks);

    sb_json_ok([
        'block' => sb_normalize_block_record($block),
    ]);
}

if ($action === 'block.update') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $block = sb_find_block($id);
    if (!$block) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $page = sb_find_page((int)($block['pageId'] ?? 0));
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $contentRaw = $_POST['content'] ?? null;
    $propsRaw = $_POST['props'] ?? null;
    $typeRaw = $_POST['type'] ?? null;

    $newContent = null;
    $newProps = null;
    $newType = null;

    if ($contentRaw !== null) {
        if (is_array($contentRaw)) {
            $newContent = $contentRaw;
        } else {
            $decoded = json_decode((string)$contentRaw, true);
            if (!is_array($decoded)) {
                sb_json_error('BAD_CONTENT_JSON', 422);
            }
            $newContent = $decoded;
        }
    }

    if ($propsRaw !== null) {
        if (is_array($propsRaw)) {
            $newProps = $propsRaw;
        } else {
            $decoded = json_decode((string)$propsRaw, true);
            if (!is_array($decoded)) {
                sb_json_error('BAD_PROPS_JSON', 422);
            }
            $newProps = $decoded;
        }
    }

    if ($typeRaw !== null) {
        $newType = trim((string)$typeRaw);
        if ($newType === '') {
            sb_json_error('TYPE_REQUIRED', 422);
        }
    }

    $blocks = sb_read_blocks();
    $updated = null;

    foreach ($blocks as &$b) {
        if ((int)($b['id'] ?? 0) === $id) {
            if ($newType !== null) {
                $b['type'] = $newType;
            }
            if ($newContent !== null) {
                $b['content'] = $newContent;
            }
            if ($newProps !== null) {
                $b['props'] = $newProps;
            }

            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();

            $updated = $b;
            break;
        }
    }
    unset($b);

    if (!$updated) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    sb_write_blocks($blocks);

    sb_json_ok([
        'block' => sb_normalize_block_record($updated),
    ]);
}

if ($action === 'block.delete') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $block = sb_find_block($id);
    if (!$block) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $page = sb_find_page((int)($block['pageId'] ?? 0));
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $before = count($blocks);

    $blocks = array_values(array_filter($blocks, static function ($b) use ($id) {
        return (int)($b['id'] ?? 0) !== $id;
    }));

    if (count($blocks) === $before) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    sb_write_blocks($blocks);
    sb_json_ok();
}

if ($action === 'block.duplicate') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }

    $src = sb_find_block($id);
    if (!$src) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $pageId = (int)($src['pageId'] ?? 0);
    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $srcSort = (int)($src['sort'] ?? 500);

    foreach ($blocks as &$b) {
        if (
            (int)($b['pageId'] ?? 0) === $pageId
            && (int)($b['sort'] ?? 0) > $srcSort
        ) {
            $b['sort'] = (int)($b['sort'] ?? 0) + 10;
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($b);

    $copy = $src;
    $copy['id'] = sb_next_block_id($blocks);
    $copy['sort'] = $srcSort + 10;
    $copy['createdBy'] = (int)$USER->GetID();
    $copy['createdAt'] = date('c');
    $copy['updatedAt'] = date('c');
    $copy['updatedBy'] = (int)$USER->GetID();

    $blocks[] = $copy;
    sb_write_blocks($blocks);

    sb_json_ok([
        'block' => sb_normalize_block_record($copy),
    ]);
}

if ($action === 'block.move') {
    $id = (int)($_POST['id'] ?? 0);
    $dir = trim((string)($_POST['dir'] ?? ''));

    if ($id <= 0) {
        sb_json_error('ID_REQUIRED', 422);
    }
    if ($dir !== 'up' && $dir !== 'down') {
        sb_json_error('DIR_REQUIRED', 422);
    }

    $block = sb_find_block($id);
    if (!$block) {
        sb_json_error('BLOCK_NOT_FOUND', 404);
    }

    $pageId = (int)($block['pageId'] ?? 0);
    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();

    $siblings = array_values(array_filter($blocks, static function ($b) use ($pageId) {
        return (int)($b['pageId'] ?? 0) === $pageId;
    }));

    usort($siblings, static function ($a, $b) {
        $sortCmp = (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500);
        if ($sortCmp !== 0) {
            return $sortCmp;
        }
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    $pos = null;
    for ($i = 0, $cnt = count($siblings); $i < $cnt; $i++) {
        if ((int)($siblings[$i]['id'] ?? 0) === $id) {
            $pos = $i;
            break;
        }
    }

    if ($pos === null) {
        sb_json_ok();
    }

    if ($dir === 'up' && $pos === 0) {
        sb_json_ok();
    }

    if ($dir === 'down' && $pos === count($siblings) - 1) {
        sb_json_ok();
    }

    $swapPos = ($dir === 'up') ? $pos - 1 : $pos + 1;

    $idA = (int)$siblings[$pos]['id'];
    $idB = (int)$siblings[$swapPos]['id'];
    $sortA = (int)($siblings[$pos]['sort'] ?? 500);
    $sortB = (int)($siblings[$swapPos]['sort'] ?? 500);

    foreach ($blocks as &$b) {
        $bid = (int)($b['id'] ?? 0);

        if ($bid === $idA) {
            $b['sort'] = $sortB;
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }

        if ($bid === $idB) {
            $b['sort'] = $sortA;
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($b);

    sb_write_blocks($blocks);
    sb_json_ok();
}

if ($action === 'block.reorder') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    if ($pageId <= 0) {
        sb_json_error('PAGE_ID_REQUIRED', 422);
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        sb_json_error('PAGE_NOT_FOUND', 404);
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $orderRaw = $_POST['order'] ?? null;
    if ($orderRaw === null) {
        sb_json_error('ORDER_REQUIRED', 422);
    }

    if (is_array($orderRaw)) {
        $order = $orderRaw;
    } else {
        $order = json_decode((string)$orderRaw, true);
        if (!is_array($order)) {
            sb_json_error('BAD_ORDER_JSON', 422);
        }
    }

    $orderIds = [];
    foreach ($order as $item) {
        $bid = (int)$item;
        if ($bid > 0) {
            $orderIds[] = $bid;
        }
    }

    $pageBlocks = sb_blocks_for_page($pageId);
    $pageBlockIds = [];
    foreach ($pageBlocks as $b) {
        $pageBlockIds[(int)($b['id'] ?? 0)] = true;
    }

    foreach ($orderIds as $bid) {
        if (!isset($pageBlockIds[$bid])) {
            sb_json_error('BLOCK_NOT_IN_PAGE', 422, ['blockId' => $bid]);
        }
    }

    $missing = array_diff(array_keys($pageBlockIds), $orderIds);
    if (!empty($missing)) {
        foreach ($missing as $bid) {
            $orderIds[] = (int)$bid;
        }
    }

    $sortMap = [];
    $sort = 10;
    foreach ($orderIds as $bid) {
        $sortMap[(int)$bid] = $sort;
        $sort += 10;
    }

    $blocks = sb_read_blocks();
    foreach ($blocks as &$b) {
        $bid = (int)($b['id'] ?? 0);
        if ((int)($b['pageId'] ?? 0) === $pageId && isset($sortMap[$bid])) {
            $b['sort'] = $sortMap[$bid];
            $b['updatedAt'] = date('c');
            $b['updatedBy'] = (int)$USER->GetID();
        }
    }
    unset($b);

    sb_write_blocks($blocks);

    sb_json_ok([
        'blocks' => array_map('sb_normalize_block_record', sb_blocks_for_page($pageId)),
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'block',
    'action' => $action,
]);


---

3. Что теперь готово по block

После вставки у тебя должны работать:

block.list

block.create

block.update

block.delete

block.duplicate

block.move

block.reorder



---

4. В каком формате можно вызывать block.update

Сейчас block.update понимает оба варианта:

Вариант 1 — как массив

Если фронт шлёт обычный POST-массив:

content[html]=<p>Текст</p>
props[className]=hero

Вариант 2 — как JSON-строка

Если фронт шлёт:

content: "{\"html\":\"<p>Текст</p>\"}"
props: "{\"className\":\"hero\"}"

Это сделано специально, чтобы не ломать старый фронт.


---

5. Что проверить руками

Проверь по порядку:

1. block.create для text


2. block.list


3. block.update содержимого


4. block.duplicate


5. block.move


6. block.reorder


7. block.delete




---

6. На что обратить внимание

Сейчас block.create создаёт стандартный content по типу блока. Это временно правильно, потому что редактор потом всё равно будет сохранять конкретные данные через block.update.


---

7. Следующий правильный шаг

Теперь у тебя backend уже начинает оживать. Логично идти дальше в таком порядке:

1. menu


2. access


3. template


4. layout


5. file / Disk



Я бы следующим делом делал menu-пакет, потому что он обычно проще Disk и быстро закрывает навигацию сайта.

Напиши: давай menu-пакет