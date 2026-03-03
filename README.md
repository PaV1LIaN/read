if ($action === 'site.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    sb_require_viewer($siteId);

    $sites = sb_read_sites();
    foreach ($sites as $s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            echo json_encode(['ok'=>true,'site'=>$s], JSON_UNESCAPED_UNICODE);
            exit;
        }
    }

    http_response_code(404);
    echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
    exit;
}
