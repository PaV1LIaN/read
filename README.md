Отлично. Начинаем с api.php.

Ниже даю готовые куски, которые нужно вставить.


---

1. Добавь helpers для layouts.json

Найди блок с JSON storage, где у тебя есть:

function sb_read_templates(): array { ... }
function sb_write_templates(array $templates): void { ... }

И сразу после них вставь это:

function sb_read_layouts(): array { return sb_read_json_file('layouts.json'); }
function sb_write_layouts(array $layouts): void { sb_write_json_file('layouts.json', $layouts, 'Cannot open layouts.json'); }

function sb_layout_empty_record(int $siteId): array {
    return [
        'siteId' => $siteId,
        'zones' => [
            'header' => [],
            'footer' => [],
            'left'   => [],
            'right'  => [],
        ],
    ];
}

function sb_layout_find_record(array $all, int $siteId): ?array {
    foreach ($all as $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) return $r;
    }
    return null;
}

function sb_layout_upsert_record(array &$all, int $siteId, array $record): void {
    $found = false;
    foreach ($all as $i => $r) {
        if ((int)($r['siteId'] ?? 0) === $siteId) {
            $all[$i] = $record;
            $found = true;
            break;
        }
    }
    if (!$found) $all[] = $record;
}

function sb_layout_ensure_record(int $siteId): array {
    $all = sb_read_layouts();
    $rec = sb_layout_find_record($all, $siteId);
    if ($rec) return $rec;

    $rec = sb_layout_empty_record($siteId);
    $all[] = $rec;
    sb_write_layouts($all);
    return $rec;
}

function sb_layout_valid_zone(string $zone): bool {
    return in_array($zone, ['header', 'footer', 'left', 'right'], true);
}

function sb_layout_zone_blocks(array $record, string $zone): array {
    $zones = $record['zones'] ?? [];
    $blocks = $zones[$zone] ?? [];
    return is_array($blocks) ? $blocks : [];
}

function sb_layout_zone_set(array &$record, string $zone, array $blocks): void {
    if (!isset($record['zones']) || !is_array($record['zones'])) $record['zones'] = [];
    $record['zones'][$zone] = array_values($blocks);
}

function sb_layout_next_block_id(array $record): int {
    $max = 0;
    $zones = is_array($record['zones'] ?? null) ? $record['zones'] : [];
    foreach ($zones as $zoneBlocks) {
        if (!is_array($zoneBlocks)) continue;
        foreach ($zoneBlocks as $b) {
            $max = max($max, (int)($b['id'] ?? 0));
        }
    }
    return $max + 1;
}

function sb_layout_next_sort(array $blocks): int {
    $max = 0;
    foreach ($blocks as $b) $max = max($max, (int)($b['sort'] ?? 0));
    return $max + 10;
}

function sb_layout_find_block(array $record, string $zone, int $blockId): ?array {
    $blocks = sb_layout_zone_blocks($record, $zone);
    foreach ($blocks as $b) {
        if ((int)($b['id'] ?? 0) === $blockId) return $b;
    }
    return null;
}


---

2. В site.create добавь layout settings

Найди внутри:

$site = [
    'id' => $id,
    ...
    'settings' => [
        'containerWidth' => 1100,
        'accent' => '#2563eb',
        'logoFileId' => 0,
    ],
];

И замени на:

$site = [
    'id' => $id,
    'name' => $name,
    'slug' => $slug,
    'createdBy' => (int)$USER->GetID(),
    'createdAt' => date('c'),
    'diskFolderId' => 0,
    'topMenuId' => 0,
    'settings' => [
        'containerWidth' => 1100,
        'accent' => '#2563eb',
        'logoFileId' => 0,
    ],
    'layout' => [
        'showHeader' => true,
        'showFooter' => true,
        'showLeft' => false,
        'showRight' => false,
        'leftWidth' => 260,
        'rightWidth' => 260,
    ],
];


---

3. Добавь actions для layout

Теперь самое главное.
Вставь этот большой блок перед самым последним:

http_response_code(400);
echo json_encode(['ok' => false, 'error' => 'UNKNOWN_ACTION', 'action' => $action], JSON_UNESCAPED_UNICODE);

Вот код:

if ($action === 'layout.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_viewer($siteId);

    $site = sb_find_site($siteId);
    if (!$site) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $record = sb_layout_ensure_record($siteId);

    echo json_encode([
        'ok' => true,
        'layout' => $site['layout'] ?? [
            'showHeader' => true,
            'showFooter' => true,
            'showLeft' => false,
            'showRight' => false,
            'leftWidth' => 260,
            'rightWidth' => 260,
        ],
        'zones' => $record['zones'] ?? [
            'header' => [],
            'footer' => [],
            'left' => [],
            'right' => [],
        ],
    ], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'layout.updateSettings') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_admin($siteId);

    $sites = sb_read_sites();
    $found = false;

    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) !== $siteId) continue;

        $layout = is_array($s['layout'] ?? null) ? $s['layout'] : [];

        if (array_key_exists('showHeader', $_POST)) {
            $layout['showHeader'] = in_array((string)$_POST['showHeader'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('showFooter', $_POST)) {
            $layout['showFooter'] = in_array((string)$_POST['showFooter'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('showLeft', $_POST)) {
            $layout['showLeft'] = in_array((string)$_POST['showLeft'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('showRight', $_POST)) {
            $layout['showRight'] = in_array((string)$_POST['showRight'], ['1', 'true', 'Y'], true);
        }
        if (array_key_exists('leftWidth', $_POST)) {
            $w = (int)$_POST['leftWidth'];
            if ($w < 160) $w = 160;
            if ($w > 500) $w = 500;
            $layout['leftWidth'] = $w;
        }
        if (array_key_exists('rightWidth', $_POST)) {
            $w = (int)$_POST['rightWidth'];
            if ($w < 160) $w = 160;
            if ($w > 500) $w = 500;
            $layout['rightWidth'] = $w;
        }

        $layout += [
            'showHeader' => true,
            'showFooter' => true,
            'showLeft' => false,
            'showRight' => false,
            'leftWidth' => 260,
            'rightWidth' => 260,
        ];

        $s['layout'] = $layout;
        $s['updatedAt'] = date('c');
        $s['updatedBy'] = (int)$USER->GetID();
        $found = true;
        break;
    }
    unset($s);

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_sites($sites);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'layout.block.list') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $zone = trim((string)($_POST['zone'] ?? ''));

    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if (!sb_layout_valid_zone($zone)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ZONE_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_viewer($siteId);

    $record = sb_layout_ensure_record($siteId);
    $blocks = sb_layout_zone_blocks($record, $zone);
    usort($blocks, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    echo json_encode(['ok'=>true,'blocks'=>$blocks], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'layout.block.create') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $zone = trim((string)($_POST['zone'] ?? ''));
    $type = trim((string)($_POST['type'] ?? 'text'));

    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if (!sb_layout_valid_zone($zone)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ZONE_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if (!in_array($type, ['text','image','button','heading','columns2','gallery','spacer','card','cards'], true)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'TYPE_NOT_SUPPORTED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_editor($siteId);

    $all = sb_read_layouts();
    $record = sb_layout_find_record($all, $siteId);
    if (!$record) $record = sb_layout_empty_record($siteId);

    $blocks = sb_layout_zone_blocks($record, $zone);
    $id = sb_layout_next_block_id($record);
    $sort = sb_layout_next_sort($blocks);

    $content = [];

    if ($type === 'text') {
        $content = ['text' => (string)($_POST['text'] ?? '')];

    } elseif ($type === 'image') {
        $fileId = (int)($_POST['fileId'] ?? 0);
        $alt = (string)($_POST['alt'] ?? '');

        if ($fileId <= 0) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if (!sb_disk_file_belongs_to_site($siteId, $fileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = ['fileId' => $fileId, 'alt' => $alt];

    } elseif ($type === 'heading') {
        $text = trim((string)($_POST['text'] ?? ''));
        $level = strtolower(trim((string)($_POST['level'] ?? 'h2')));
        $align = strtolower(trim((string)($_POST['align'] ?? 'left')));

        if ($text === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if (!in_array($level, ['h1','h2','h3'], true)) $level = 'h2';
        if (!in_array($align, ['left','center','right'], true)) $align = 'left';

        $content = ['text' => $text, 'level' => $level, 'align' => $align];

    } elseif ($type === 'columns2') {
        $left  = (string)($_POST['left'] ?? '');
        $right = (string)($_POST['right'] ?? '');
        $ratio = trim((string)($_POST['ratio'] ?? '50-50'));
        if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

        $content = [
            'left' => $left,
            'right' => $right,
            'ratio' => $ratio,
        ];

    } elseif ($type === 'gallery') {
        $columns = (int)($_POST['columns'] ?? 3);
        if (!in_array($columns, [2,3,4], true)) $columns = 3;

        $imagesJson = (string)($_POST['images'] ?? '[]');
        $images = json_decode($imagesJson, true);
        if (!is_array($images)) $images = [];

        $clean = [];
        foreach ($images as $it) {
            if (!is_array($it)) continue;
            $fid = (int)($it['fileId'] ?? 0);
            if ($fid <= 0) continue;

            if (!sb_disk_file_belongs_to_site($siteId, $fid)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$fid], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $clean[] = [
                'fileId' => $fid,
                'alt' => (string)($it['alt'] ?? ''),
            ];
        }

        if (count($clean) === 0) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'IMAGES_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = [
            'columns' => $columns,
            'images' => $clean,
        ];

    } elseif ($type === 'spacer') {
        $height = (int)($_POST['height'] ?? 40);
        if ($height < 10) $height = 10;
        if ($height > 200) $height = 200;

        $line = (string)($_POST['line'] ?? '0');
        $line = ($line === '1' || $line === 'true');

        $content = ['height' => $height, 'line' => $line];

    } elseif ($type === 'card') {
        $title = trim((string)($_POST['title'] ?? ''));
        $text  = (string)($_POST['text'] ?? '');
        $imageFileId = (int)($_POST['imageFileId'] ?? 0);
        $buttonText = trim((string)($_POST['buttonText'] ?? ''));
        $buttonUrl  = trim((string)($_POST['buttonUrl'] ?? ''));

        if ($title === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TITLE_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
            exit;
        }

        if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = [
            'title' => $title,
            'text' => $text,
            'imageFileId' => $imageFileId,
            'buttonText' => $buttonText,
            'buttonUrl' => $buttonUrl,
        ];

    } elseif ($type === 'cards') {
        $columns = (int)($_POST['columns'] ?? 3);
        if (!in_array($columns, [2,3,4], true)) $columns = 3;

        $itemsJson = (string)($_POST['items'] ?? '[]');
        $items = json_decode($itemsJson, true);
        if (!is_array($items)) $items = [];

        $clean = [];
        foreach ($items as $it) {
            if (!is_array($it)) continue;

            $title = trim((string)($it['title'] ?? ''));
            $text  = (string)($it['text'] ?? '');
            if ($title === '') continue;

            $imageFileId = (int)($it['imageFileId'] ?? 0);
            if ($imageFileId > 0 && !sb_disk_file_belongs_to_site($siteId, $imageFileId)) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'FILE_NOT_IN_SITE_FOLDER','fileId'=>$imageFileId], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $buttonText = trim((string)($it['buttonText'] ?? ''));
            $buttonUrl  = trim((string)($it['buttonUrl'] ?? ''));

            if ($buttonUrl !== '' && !(preg_match('~^https?://~i', $buttonUrl) || str_starts_with($buttonUrl, '/'))) {
                http_response_code(422);
                echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
                exit;
            }

            $clean[] = [
                'title' => $title,
                'text' => $text,
                'imageFileId' => $imageFileId,
                'buttonText' => $buttonText,
                'buttonUrl' => $buttonUrl,
            ];
        }

        if (count($clean) === 0) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'ITEMS_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = [
            'columns' => $columns,
            'items' => $clean,
        ];

    } else {
        $text = trim((string)($_POST['text'] ?? ''));
        $url  = trim((string)($_POST['url'] ?? ''));
        $variant = strtolower(trim((string)($_POST['variant'] ?? 'primary')));

        if ($text === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TEXT_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if ($url === '') {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_REQUIRED'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if (!in_array($variant, ['primary','secondary'], true)) $variant = 'primary';

        if (!(preg_match('~^https?://~i', $url) || str_starts_with($url, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = ['text' => $text, 'url' => $url, 'variant' => $variant];
    }

    $block = [
        'id' => $id,
        'type' => $type,
        'sort' => $sort,
        'content' => $content,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
    ];

    $blocks[] = $block;
    sb_layout_zone_set($record, $zone, $blocks);
    sb_layout_upsert_record($all, $siteId, $record);
    sb_write_layouts($all);

    echo json_encode(['ok'=>true,'block'=>$block], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'layout.block.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $zone = trim((string)($_POST['zone'] ?? ''));
    $id = (int)($_POST['id'] ?? 0);

    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if (!sb_layout_valid_zone($zone)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ZONE_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_editor($siteId);

    $all = sb_read_layouts();
    $record = sb_layout_find_record($all, $siteId);
    if (!$record) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'LAYOUT_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $blocks = sb_layout_zone_blocks($record, $zone);
    $before = count($blocks);
    $blocks = array_values(array_filter($blocks, fn($b) => (int)($b['id'] ?? 0) !== $id));

    if (count($blocks) === $before) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'BLOCK_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_layout_zone_set($record, $zone, $blocks);
    sb_layout_upsert_record($all, $siteId, $record);
    sb_write_layouts($all);

    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($action === 'layout.block.move') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $zone = trim((string)($_POST['zone'] ?? ''));
    $id = (int)($_POST['id'] ?? 0);
    $dir = (string)($_POST['dir'] ?? '');

    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if (!sb_layout_valid_zone($zone)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ZONE_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    if ($dir !== 'up' && $dir !== 'down') {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'DIR_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_editor($siteId);

    $all = sb_read_layouts();
    $record = sb_layout_find_record($all, $siteId);
    if (!$record) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'LAYOUT_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $blocks = sb_layout_zone_blocks($record, $zone);
    usort($blocks, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    $pos = null;
    for ($i=0; $i<count($blocks); $i++) {
        if ((int)($blocks[$i]['id'] ?? 0) === $id) { $pos = $i; break; }
    }

    if ($pos === null) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'BLOCK_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if ($dir === 'up' && $pos === 0) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir === 'down' && $pos === count($blocks)-1) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }

    $swap = ($dir === 'up') ? $pos - 1 : $pos + 1;

    $sortA = (int)($blocks[$pos]['sort'] ?? 500);
    $sortB = (int)($blocks[$swap]['sort'] ?? 500);

    $blocks[$pos]['sort'] = $sortB;
    $blocks[$pos]['updatedAt'] = date('c');
    $blocks[$swap]['sort'] = $sortA;
    $blocks[$swap]['updatedAt'] = date('c');

    sb_layout_zone_set($record, $zone, $blocks);
    sb_layout_upsert_record($all, $siteId, $record);
    sb_write_layouts($all);

    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}


---

4. Создай файл layouts.json

Создай:

/upload/sitebuilder/layouts.json

С содержимым:

[]


---

Что проверять после этого

Сначала вручную через AJAX/POST или через будущий UI:

Проверка 1

layout.get

Должен вернуть:

layout

zones


Проверка 2

layout.block.create Например для header

Проверка 3

layout.block.list Должен показать созданный блок


---

Важно

Пока здесь нет:

layout.block.update


Я специально не стал втискивать всё сразу, чтобы не перегрузить и не наломать.
Сейчас важнее завести базу и убедиться, что:

layouts.json читается,

зоны создаются,

блоки добавляются/листятся/удаляются/двигаются.


Следующий шаг — сделать layout.php и уже оттуда быстро проверить всё руками.