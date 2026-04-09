Причина найдена точно.

У тебя lib/access.php перезаписан кодом handler-а.

Смотри на начало файла из диагностики:

<?php

global $USER;

if ($action === 'access.list') {

Такого в lib/access.php быть не должно вообще.
В lib/access.php должны быть только функции sb_user_access_code, sb_get_role, sb_require_*, а не обработка action.

Из-за этого при каждом запросе bootstrap.php подключает lib/access.php, а тот сразу пытается обработать action и возвращает:

{"ok":false,"error":"NOT_MOVED_YET","handler":"access","action":"ping"}


---

Что нужно сделать

1. Полностью замени /local/sitebuilder/lib/access.php на этот код

<?php

require_once __DIR__ . '/json.php';
require_once __DIR__ . '/response.php';

if (!function_exists('sb_user_access_code')) {
    function sb_user_access_code(): string
    {
        global $USER;
        return 'U' . (int)$USER->GetID();
    }
}

if (!function_exists('sb_get_role')) {
    function sb_get_role(int $siteId, string $accessCode): ?string
    {
        $access = sb_read_access();

        foreach ($access as $row) {
            if (
                (int)($row['siteId'] ?? 0) === $siteId
                && (string)($row['accessCode'] ?? '') === $accessCode
            ) {
                return (string)($row['role'] ?? '');
            }
        }

        return null;
    }
}

if (!function_exists('sb_role_rank')) {
    function sb_role_rank(?string $role): int
    {
        switch ((string)$role) {
            case 'VIEWER':
                return 1;
            case 'EDITOR':
                return 2;
            case 'ADMIN':
                return 3;
            case 'OWNER':
                return 4;
            default:
                return 0;
        }
    }
}

if (!function_exists('sb_require_site_role')) {
    function sb_require_site_role(int $siteId, int $minRank): void
    {
        $role = sb_get_role($siteId, sb_user_access_code());

        if (sb_role_rank($role) < $minRank) {
            sb_json_error('ACCESS_DENIED', 403);
        }
    }
}

if (!function_exists('sb_require_owner')) {
    function sb_require_owner(int $siteId): void
    {
        sb_require_site_role($siteId, 4);
    }
}

if (!function_exists('sb_require_admin')) {
    function sb_require_admin(int $siteId): void
    {
        sb_require_site_role($siteId, 3);
    }
}

if (!function_exists('sb_require_editor')) {
    function sb_require_editor(int $siteId): void
    {
        sb_require_site_role($siteId, 2);
    }
}

if (!function_exists('sb_require_viewer')) {
    function sb_require_viewer(int $siteId): void
    {
        sb_require_site_role($siteId, 1);
    }
}


---

2. Верни /local/sitebuilder/api.php в нормальный вид

<?php
ini_set('display_errors', '1');
error_reporting(E_ALL);

require_once __DIR__ . '/api/index.php';


---

3. Сбрось opcache ещё раз

Открой:

/local/sitebuilder/opcache_reset.php

Должно быть:

bool(true)


---

4. Проверь Ping

Теперь Ping должен вернуть что-то вроде:

{
  "ok": true,
  "pong": true,
  "handler": "common",
  "file": "/srv/bx/docroot/local/sitebuilder/api/handlers/common.php",
  "userId": 1,
  "login": "admin"
}


---

5. Потом проверь Site list

Ожидаемый ответ:

{
  "ok": true,
  "sites": [],
  "handler": "site",
  "file": "/srv/bx/docroot/local/sitebuilder/api/handlers/site.php"
}

или список сайтов.


---

Что именно произошло

Ты где-то при копировании файлов вставил код api/handlers/access.php в lib/access.php.

То есть:

api/handlers/access.php — должен быть action-handler

lib/access.php — должен быть набор функций доступа


А у тебя в lib/access.php оказался handler.


---

Что делать дальше

Сейчас не трогай больше ничего лишнего.

Сделай только:

1. замени lib/access.php


2. верни api.php


3. сбрось opcache


4. проверь Ping


5. проверь Site list



И пришли мне два ответа:

Ping

Site list


После этого уже спокойно продолжим сборку проекта без этой путаницы.