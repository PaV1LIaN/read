Отлично. Тогда сначала добавляем layout.block.update в api.php.

Вставь этот блок рядом с остальными layout.block.* action'ами, лучше всего после layout.block.create и до layout.block.delete.

if ($action === 'layout.block.update') {
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

            $b['content']['text'] = $text;
            $b['content']['level'] = $level;
            $b['content']['align'] = $align;

        } elseif ($type === 'columns2') {
            $left  = (string)($_POST['left'] ?? '');
            $right = (string)($_POST['right'] ?? '');
            $ratio = trim((string)($_POST['ratio'] ?? '50-50'));

            if (!in_array($ratio, ['50-50','33-67','67-33'], true)) $ratio = '50-50';

            $b['content']['left'] = $left;
            $b['content']['right'] = $right;
            $b['content']['ratio'] = $ratio;

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

            $b['content']['columns'] = $columns;
            $b['content']['images'] = $clean;

        } elseif ($type === 'spacer') {
            $height = (int)($_POST['height'] ?? 40);
            if ($height < 10) $height = 10;
            if ($height > 200) $height = 200;

            $line = (string)($_POST['line'] ?? '0');
            $line = ($line === '1' || $line === 'true');

            $b['content']['height'] = $height;
            $b['content']['line'] = $line;

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

            $b['content']['title'] = $title;
            $b['content']['text'] = $text;
            $b['content']['imageFileId'] = $imageFileId;
            $b['content']['buttonText'] = $buttonText;
            $b['content']['buttonUrl'] = $buttonUrl;

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

            $b['content']['columns'] = $columns;
            $b['content']['items'] = $clean;

        } else {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'TYPE_NOT_SUPPORTED'], JSON_UNESCAPED_UNICODE);
            exit;
        }

        $b['updatedAt'] = date('c');
        $b['updatedBy'] = (int)$USER->GetID();
        $found = true;
        break;
    }
    unset($b);

    if (!$found) {
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

Как быстро проверить

Через твой временный layout_test.php можно потом добавить вызов layout.block.update, но сейчас достаточно убедиться, что api.php сохраняется без PHP-ошибок.

Следующим сообщением пришлю полный готовый layout.php.