if ($action === 'page.setStatus') {
    $id = (int)($_POST['id'] ?? 0);
    $status = strtolower(trim((string)($_POST['status'] ?? '')));

    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok' => false, 'error' => 'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if (!in_array($status, ['draft', 'published'], true)) {
        http_response_code(422);
        echo json_encode(['ok' => false, 'error' => 'STATUS_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page($id);
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok' => false, 'error' => 'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $pages = sb_read_pages();
    $found = false;

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) !== $id) continue;

        $p['status'] = $status;
        $p['updatedAt'] = date('c');
        $p['updatedBy'] = (int)$USER->GetID();

        if ($status === 'published') {
            if (empty($p['publishedAt'])) {
                $p['publishedAt'] = date('c');
            }
        } else {
            $p['publishedAt'] = '';
        }

        $found = true;
        break;
    }
    unset($p);

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok' => false, 'error' => 'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_pages($pages);

    echo json_encode([
        'ok' => true,
        'id' => $id,
        'status' => $status,
    ], JSON_UNESCAPED_UNICODE);
    exit;
}