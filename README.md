{
    "ok": true,
    "apiFile": "\/srv\/bx\/docroot\/local\/sitebuilder\/api.php",
    "time": "2026-04-09T15:53:36+03:00",
    "files": {
        "api_bootstrap": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/bootstrap.php",
            "exists": true,
            "md5": "6f925665285b9e86fe2fd074c80dbc79",
            "size": 1169,
            "head": "<?php\n\ndefine('NO_KEEP_STATISTIC', true);\ndefine('NO_AGENT_STATISTIC', true);\ndefine('DisableEventsCheck', true);\n\nrequire $_SERVER['DOCUMENT_ROOT'] . '\/bitrix\/modules\/main\/include\/prolog_before.php';\n\nuse Bitrix\\Main\\Loader;\nuse Bitrix\\Disk\\Storage;\nuse Bitrix\\Disk\\Folder;\nuse Bitrix\\Disk\\File;\nuse Bitrix\\Disk\\Driver;\n\nglobal $USER;\n\nheader('Content-Type: application\/json; charset=UTF-8');\n\nif (!$USER->IsAuthorized()) {\n    http_response_code(401);\n    echo json_encode(['ok' => false, 'error' => 'NOT_AUTHORIZED'], JSON_UNESCAPED_UNICODE);\n    exit;\n}\n\nif ($_SERVER['REQUEST_METHOD'] !== 'POST'"
        },
        "lib_json": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/lib\/json.php",
            "exists": true,
            "md5": "ad5f057f0653755a8ba0205e5c74dfaa",
            "size": 3838,
            "head": "<?php\n\nif (!function_exists('sb_data_path')) {\n    function sb_data_path(string $file): string\n    {\n        return $_SERVER['DOCUMENT_ROOT'] . '\/upload\/sitebuilder\/' . $file;\n    }\n}\n\nif (!function_exists('sb_read_json_file')) {\n    function sb_read_json_file(string $file): array\n    {\n        $path = sb_data_path($file);\n        if (!file_exists($path)) {\n            return [];\n        }\n\n        $fp = fopen($path, 'rb');\n        if (!$fp) {\n            return [];\n        }\n\n        $raw = '';\n        if (flock($fp, LOCK_SH)) {\n            $raw = stream_get_contents($fp);\n            flock($"
        },
        "lib_response": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/lib\/response.php",
            "exists": true,
            "md5": "08045556cc3e778cdc5a57e5cef4f83c",
            "size": 683,
            "head": "<?php\n\nif (!function_exists('sb_json_response')) {\n    function sb_json_response(array $payload, int $status = 200): void\n    {\n        http_response_code($status);\n        echo json_encode($payload, JSON_UNESCAPED_UNICODE);\n        exit;\n    }\n}\n\nif (!function_exists('sb_json_ok')) {\n    function sb_json_ok(array $data = []): void\n    {\n        sb_json_response(array_merge(['ok' => true], $data), 200);\n    }\n}\n\nif (!function_exists('sb_json_error')) {\n    function sb_json_error(string $error, int $status = 400, array $extra = []): void\n    {\n        sb_json_response(array_merge([\n            "
        },
        "lib_access": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/lib\/access.php",
            "exists": true,
            "md5": "c23f4eebb15ce71d171fcce71d1c0478",
            "size": 3695,
            "head": "<?php\n\nglobal $USER;\n\nif ($action === 'access.list') {\n    $siteId = (int)($_POST['siteId'] ?? 0);\n    if ($siteId <= 0) {\n        sb_json_error('SITE_ID_REQUIRED', 422);\n    }\n\n    sb_require_admin($siteId);\n\n    $rows = sb_access_rows_for_site($siteId);\n\n    usort($rows, static function ($a, $b) {\n        $roleCmp = sb_role_rank((string)($b['role'] ?? '')) <=> sb_role_rank((string)($a['role'] ?? ''));\n        if ($roleCmp !== 0) {\n            return $roleCmp;\n        }\n\n        return strcmp((string)($a['accessCode'] ?? ''), (string)($b['accessCode'] ?? ''));\n    });\n\n    sb_json_ok([\n      "
        },
        "lib_helpers": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/lib\/helpers.php",
            "exists": true,
            "md5": "ce4df5ebaf1e08a64db3da3945471a18",
            "size": 18030,
            "head": "<?php\n\nrequire_once __DIR__ . '\/json.php';\n\nif (!function_exists('sb_slugify')) {\n    function sb_slugify(string $name): string\n    {\n        $slug = \\CUtil::translit($name, 'ru', [\n            'replace_space' => '-',\n            'replace_other' => '-',\n            'change_case' => 'L',\n            'delete_repeat_replace' => true,\n            'use_google' => false,\n        ]);\n\n        $slug = trim($slug, '-');\n        return $slug !== '' ? $slug : 'item';\n    }\n}\n\nif (!function_exists('sb_site_exists')) {\n    function sb_site_exists(int $siteId): bool\n    {\n        foreach (sb_read_sites() as"
        },
        "lib_disk": {
            "path": "\/srv\/bx\/docroot\/local\/sitebuilder\/lib\/disk.php",
            "exists": true,
            "md5": "0ea05aad362b50af9de0ed3e98af43fb",
            "size": 9564,
            "head": "<?php\n\nrequire_once __DIR__ . '\/json.php';\nrequire_once __DIR__ . '\/helpers.php';\n\nuse Bitrix\\Main\\Loader;\nuse Bitrix\\Disk\\Driver;\nuse Bitrix\\Disk\\Storage;\nuse Bitrix\\Disk\\Folder;\nuse Bitrix\\Disk\\File;\nuse Bitrix\\Disk\\Security\\FakeSecurityContext;\n\nif (!function_exists('sb_disk_require_module')) {\n    function sb_disk_require_module(): void\n    {\n        if (!Loader::includeModule('disk')) {\n            throw new RuntimeException('Disk module is not installed');\n        }\n    }\n}\n\nif (!function_exists('sb_disk_security_context')) {\n    function sb_disk_security_context()\n    {\n        global $"
        }
    }
}
