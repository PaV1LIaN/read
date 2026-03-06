if ($action === 'block.duplicate') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $src = sb_find_block($id);
    if (!$src) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page((int)($src['pageId'] ?? 0));
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();

    $maxId = 0;
    foreach ($blocks as $b) {
        $maxId = max($maxId, (int)($b['id'] ?? 0));
    }
    $newId = $maxId + 1;

    $srcSort = (int)($src['sort'] ?? 500);

    // Сдвигаем все блоки ниже, чтобы вставить копию сразу после исходного
    foreach ($blocks as &$b) {
        if ((int)($b['pageId'] ?? 0) === (int)($src['pageId'] ?? 0) && (int)($b['sort'] ?? 0) > $srcSort) {
            $b['sort'] = (int)($b['sort'] ?? 0) + 10;
            $b['updatedAt'] = date('c');
        }
    }
    unset($b);

    $copy = $src;
    $copy['id'] = $newId;
    $copy['sort'] = $srcSort + 10;
    $copy['createdBy'] = (int)$USER->GetID();
    $copy['createdAt'] = date('c');
    $copy['updatedAt'] = date('c');

    $blocks[] = $copy;
    sb_write_blocks($blocks);

    echo json_encode(['ok'=>true,'block'=>$copy], JSON_UNESCAPED_UNICODE);
    exit;
}