Да, понял. У тебя рабочая таблица страниц называется:

sitebuilder.page

А я в PageAccessRepository.php указал:

sitebuilder.pages

Поэтому надо заменить только это место.

Старые таблицы типа:

sitebuilder_page
sitebuilder_site
sitebuilder_block
sitebuilder_site_user_access

пока не удаляй. Да, похоже, часть из них осталась от ранних вариантов разработки, но сейчас трогать их опасно, пока не проверим, какие реально используются кодом.


---

Исправь PageAccessRepository.php

Файл:

/local/sitebuilder/lib/PageAccessRepository.php

Найди метод:

public static function getPageAndParentIds(int $siteId, int $pageId): array

Внутри него сейчас есть запрос:

$stmt = $pdo->prepare("
    SELECT id, parent_id
    FROM sitebuilder.pages
    WHERE site_id = :site_id
      AND id = :page_id
    LIMIT 1
");

Замени sitebuilder.pages на sitebuilder.page:

$stmt = $pdo->prepare("
    SELECT id, parent_id
    FROM sitebuilder.page
    WHERE site_id = :site_id
      AND id = :page_id
    LIMIT 1
");


---

Полный правильный метод

Можешь заменить весь метод целиком:

public static function getPageAndParentIds(int $siteId, int $pageId): array
{
    if ($siteId <= 0 || $pageId <= 0) {
        return [];
    }

    $pdo = sb_db();

    $ids = [];
    $visited = [];
    $currentPageId = $pageId;

    for ($i = 0; $i < 100; $i++) {
        if ($currentPageId <= 0) {
            break;
        }

        if (isset($visited[$currentPageId])) {
            break;
        }

        $visited[$currentPageId] = true;

        $stmt = $pdo->prepare("
            SELECT id, parent_id
            FROM sitebuilder.page
            WHERE site_id = :site_id
              AND id = :page_id
            LIMIT 1
        ");

        $stmt->execute([
            ':site_id' => $siteId,
            ':page_id' => $currentPageId,
        ]);

        $row = $stmt->fetch(PDO::FETCH_ASSOC);

        if (!$row) {
            break;
        }

        $ids[] = (int)$row['id'];
        $currentPageId = (int)($row['parent_id'] ?? 0);
    }

    return $ids;
}


---

Ещё поправь диагностический файл, если используешь

В run_page_access.php у тебя проверка была:

to_regclass('sitebuilder.pages') AS pages

Замени на:

to_regclass('sitebuilder.page') AS page

То есть:

$stmt = $pdo->query("
    SELECT
        to_regclass('sitebuilder.page_access') AS page_access,
        to_regclass('sitebuilder.page') AS page,
        to_regclass('sitebuilder.access') AS access
");


---

После правки снова проверь pageAccess.save

Запусти консольный тест ещё раз:

(async function () {
  const siteId = Number(prompt('siteId'));
  const pageId = Number(prompt('pageId'));
  const userId = Number(prompt('ID пользователя, кому выдать доступ'));

  const fd = new FormData();

  fd.append('action', 'pageAccess.save');
  fd.append('sessid', BX.bitrix_sessid());
  fd.append('siteId', siteId);
  fd.append('pageId', pageId);
  fd.append('accessCode', 'U' + userId);
  fd.append('canView', '1');
  fd.append('canEdit', '1');
  fd.append('includeChildren', '1');

  const res = await fetch('/local/sitebuilder/api/index.php', {
    method: 'POST',
    body: fd
  });

  console.log(await res.json());
})();

Если всё нормально, должен вернуться:

{
  "ok": true,
  "data": {
    "item": {
      "siteId": 1,
      "pageId": 1,
      "accessCode": "U99",
      "canView": true,
      "canEdit": true,
      "includeChildren": true
    }
  }
}


---

Кратко: рабочие новые таблицы у нас сейчас:

sitebuilder.site
sitebuilder.page
sitebuilder.block
sitebuilder.access
sitebuilder.page_access

А sitebuilder_site, sitebuilder_page, sitebuilder_block пока считаем старым хвостом и не используем.