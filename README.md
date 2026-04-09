{
    "ok": true,
    "apiFile": "\/srv\/bx\/docroot\/local\/sitebuilder\/api.php",
    "time": "2026-04-09T15:51:10+03:00",
    "files": {
        "api_index": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/index.php",
            "exists": true,
            "md5": "3c710ee1d3975799bf47715016909afc",
            "size": 358,
            "head": "<?php\n\nrequire_once __DIR__ . '\/bootstrap.php';\n\n$action = (string)($_POST['action'] ?? '');\n\nif ($action === 'ping') {\n    require __DIR__ . '\/handlers\/common.php';\n    exit;\n}\n\nif ($action === 'site.list') {\n    require __DIR__ . '\/handlers\/site.php';\n    exit;\n}\n\nsb_json_error('UNKNOWN_ACTION', 400, [\n    'action' => $action,\n    'file' => __FILE__,\n]);"
        },
        "handler_common": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/handlers\/common.php",
            "exists": true,
            "md5": "02e1f89bb8479607ed5860c86c2d5189",
            "size": 392,
            "head": "<?php\n\nglobal $USER;\n\nif ($action === 'ping') {\n    sb_json_ok([\n        'pong' => true,\n        'handler' => 'common',\n        'file' => __FILE__,\n        'userId' => (int)$USER->GetID(),\n        'login' => (string)$USER->GetLogin(),\n        'time' => date('c'),\n    ]);\n}\n\nsb_json_error('NOT_MOVED_YET', 501, [\n    'handler' => 'common',\n    'action' => $action,\n    'file' => __FILE__,\n]);"
        },
        "handler_site": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/handlers\/site.php",
            "exists": true,
            "md5": "a05404ec14981afa3cba38651d621d72",
            "size": 904,
            "head": "if ($action === 'site.list') {\n    $sites = sb_read_sites();\n    $myCode = sb_user_access_code();\n\n    $access = sb_read_access();\n    $allowedSiteIds = [];\n\n    foreach ($access as $r) {\n        if ((string)($r['accessCode'] ?? '') === $myCode) {\n            $sid = (int)($r['siteId'] ?? 0);\n            if ($sid > 0) {\n                $allowedSiteIds[$sid] = true;\n            }\n        }\n    }\n\n  "
        },
        "handler_access": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/handlers\/access.php",
            "exists": true,
            "md5": "c8610555408d4591a759f130dde3499c",
            "size": 3903,
            "head": "<?php\n\nglobal $USER;\n\nif ($action === 'access.list') {\n    $siteId = (int)($_POST['siteId'] ?? 0);\n    if ($siteId <= 0) {\n        sb_json_error('SITE_ID_REQUIRED', 422);\n    }\n\n    sb_require_admin($siteId);\n\n    $rows = sb_access_rows_for_site($siteId);\n\n    usort($rows, static function ($a, $b) {\n        $roleCmp = sb_role_rank((string)($b['role'] ?? '')) <=> sb_role_rank((string)($a['role'] ??"
        }
    }
}
