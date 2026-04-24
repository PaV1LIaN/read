Ошибка нормальная для текущего этапа:

EMPTY_BITRIX_GROUP_ID

Группа в БД есть, но sb_find_site($siteId) возвращает сайт без поля bitrixGroupId. Это видно по твоему site.get: там нет bitrixGroupId.

Сейчас сделаем быстрый рабочий фикс прямо в site.php, чтобы site.syncAccess сам дочитывал bitrix_group_id из PostgreSQL.

В site.php замени блок site.syncAccess

Файл:

/local/sitebuilder/api/handlers/site.php

Найди блок:

if ($action === 'site.syncAccess') {
    ...
}

И замени его целиком на этот:

if ($action === 'site.syncAccess') {
    $siteId = (int)($_POST['siteId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_owner($siteId);

    $site = sb_find_site($siteId);

    if (!$site) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    /*
     * Важно:
     * sb_find_site() сейчас не возвращает bitrixGroupId,
     * поэтому дочитываем bitrix_group_id напрямую из PostgreSQL.
     */
    $bitrixGroupId = (int)(
        $site['bitrixGroupId']
        ?? $site['bitrix_group_id']
        ?? 0
    );

    if ($bitrixGroupId <= 0 && function_exists('sb_db')) {
        try {
            $pdo = sb_db();

            $st = $pdo->prepare("
                SELECT bitrix_group_id
                FROM sitebuilder.site
                WHERE id = :site_id
                LIMIT 1
            ");

            $st->execute([
                ':site_id' => $siteId,
            ]);

            $row = $st->fetch(PDO::FETCH_ASSOC);

            if ($row) {
                $bitrixGroupId = (int)($row['bitrix_group_id'] ?? 0);
            }
        } catch (Throwable $e) {
            sb_json_error('BITRIX_GROUP_ID_READ_ERROR: ' . $e->getMessage(), 500, [
                'handler' => 'site',
                'action' => 'site.syncAccess',
                'file' => __FILE__,
            ]);
        }
    }

    if ($bitrixGroupId <= 0) {
        sb_json_error('EMPTY_BITRIX_GROUP_ID', 500, [
            'handler' => 'site',
            'action' => 'site.syncAccess',
            'siteId' => $siteId,
            'site' => $site,
            'file' => __FILE__,
        ]);
    }

    $site['bitrixGroupId'] = $bitrixGroupId;
    $site['bitrix_group_id'] = $bitrixGroupId;

    if (!class_exists('SiteAccessSyncService')) {
        sb_json_error('SiteAccessSyncService.php не подключен', 500);
    }

    try {
        $result = SiteAccessSyncService::syncSiteAccessFromBitrixGroup(
            $site,
            (int)$USER->GetID(),
            true
        );

        sb_json_ok([
            'result' => $result,
            'handler' => 'site',
            'action' => 'site.syncAccess',
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => 'site.syncAccess',
            'file' => __FILE__,
        ]);
    }
}

После замены снова выполни:

fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    action: 'site.syncAccess',
    siteId: '11',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(async function (r) {
  const text = await r.text();
  console.log('STATUS:', r.status);
  console.log('TEXT:', text);
  try {
    console.log('JSON:', JSON.parse(text));
  } catch (e) {
    console.log('NOT JSON');
  }
})
.catch(console.error);

Если всё нормально, должен вернуться ok: true.

Потом уже правильно поправим нормализацию сайта, чтобы site.get тоже возвращал:

"bitrixGroupId": 2