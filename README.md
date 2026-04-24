Дальше правим site.list, чтобы список сайтов показывал не только сайты из sitebuilder.access, но и сайты, где пользователь состоит в привязанной группе Битрикс24.

Сейчас прямой доступ уже должен работать через fallback в access.php, но список сайтов всё ещё фильтруется старым способом.

1. В site.php замени блок site.list

Файл:

/local/sitebuilder/api/handlers/site.php

Найди весь блок:

if ($action === 'site.list') {
    ...
}

и замени его на этот:

if ($action === 'site.list') {
    $sites = sb_read_sites();
    $myCode = sb_user_access_code();

    $allowedSites = [];

    foreach ($sites as $site) {
        $siteId = (int)($site['id'] ?? 0);

        if ($siteId <= 0) {
            continue;
        }

        /*
         * Важно:
         * sb_get_role() теперь проверяет:
         * 1. sitebuilder.access
         * 2. fallback через группу Битрикс24
         */
        $role = sb_get_role($siteId, $myCode);

        if (sb_role_rank($role) < 1) {
            continue;
        }

        $site['currentUserRole'] = $role;
        $allowedSites[] = $site;
    }

    usort($allowedSites, static function ($a, $b) {
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    sb_json_ok([
        'sites' => $allowedSites,
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}


---

2. Что изменится

Теперь site.list будет учитывать:

sitebuilder.access → основная таблица прав
группа Битрикс24 → fallback-доступ

То есть если пользователя добавили в группу сайта как обычного участника, он должен увидеть сайт в списке даже без ручной синхронизации.

Также в каждом сайте появится поле:

"currentUserRole": "VIEWER"

или:

"currentUserRole": "EDITOR"

или:

"currentUserRole": "OWNER"


---

3. Проверка

Под пользователем, которого ты добавил в группу Битрикс24, открой:

/local/sitebuilder/index.php

Если фронт использует site.list, сайт должен появиться.

Можно проверить через консоль:

fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    action: 'site.list',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.json())
.then(console.log);

В ответе должен быть сайт с bitrixGroupId и currentUserRole.

После этого следующим шагом можно будет в списке сайтов вывести роль пользователя и кнопку «Открыть группу».