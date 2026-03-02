if ($action === 'block.update') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $blk = sb_find_block($id);
    if (!$blk) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $page = sb_find_page((int)($blk['pageId'] ?? 0));
    if (!$page) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $blocks = sb_read_blocks();
    $found = false;

    foreach ($blocks as &$b) {
        if ((int)($b['id'] ?? 0) !== $id) continue;

        $type = (string)($b['type'] ?? '');

        if ($type === 'text') {
            $b['content']['text'] = (string)($_POST['text'] ?? '');

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

            $b['content']['fileId'] = $fileId;
            $b['content']['alt'] = $alt;

        } elseif ($type === 'button') {
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

            $b['content']['text'] = $text;
            $b['content']['url'] = $url;
            $b['content']['variant'] = $variant;

        } else {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TYPE_NOT_SUPPORTED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $b['updatedAt'] = date('c');
        $found = true;
        break;
    }
    unset($b);

    if (!$found) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_write_blocks($blocks);
    echo json_encode(['ok' => true], JSON_UNESCAPED_UNICODE);
    exit;
}
