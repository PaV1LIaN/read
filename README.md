if ($action === 'block.create') {
    $pageId = (int)($_POST['pageId'] ?? 0);
    $type   = trim((string)($_POST['type'] ?? 'text'));

    if ($pageId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'PAGE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    // ДОБАВИЛИ button
    if (!in_array($type, ['text','image','button'], true)) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'TYPE_NOT_SUPPORTED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page($pageId);
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }
    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $maxId = 0;
    foreach ($blocks as $b) $maxId = max($maxId, (int)($b['id'] ?? 0));
    $id = $maxId + 1;

    $maxSort = 0;
    foreach ($blocks as $b) {
        if ((int)($b['pageId'] ?? 0) === $pageId) {
            $maxSort = max($maxSort, (int)($b['sort'] ?? 0));
        }
    }
    $sort = $maxSort + 10;

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

    } else { // button
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

        // внешний http(s) или относительный /
        if (!(preg_match('~^https?://~i', $url) || str_starts_with($url, '/'))) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'URL_BAD_FORMAT'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $content = ['text' => $text, 'url' => $url, 'variant' => $variant];
    }

    $block = [
        'id' => $id,
        'pageId' => $pageId,
        'type' => $type,
        'sort' => $sort,
        'content' => $content,
        'createdBy' => (int)$USER->GetID(),
        'createdAt' => date('c'),
        'updatedAt' => date('c'),
    ];

    $blocks[] = $block;
    sb_write_blocks($blocks);

    echo json_encode(['ok' => true, 'block' => $block], JSON_UNESCAPED_UNICODE);
    exit;
}
