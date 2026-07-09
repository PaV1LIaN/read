Да, дальше добавляем API для выдачи прав на страницы:

pageAccess.list    — получить права страницы
pageAccess.save    — выдать/обновить право
pageAccess.delete  — удалить право


---

1. Создай папку

/local/sitebuilder/api/actions/page_access/


---

2. Файл list.php

Файл:

/local/sitebuilder/api/actions/page_access/list.php

Код:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessRepository.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessService.php';

global $USER;

if (!is_object($USER) || !$USER->IsAuthorized()) {
    throw new RuntimeException('AUTH_REQUIRED');
}

$data = json_decode(file_get_contents('php://input'), true);

if (!is_array($data)) {
    $data = $_REQUEST;
}

$siteId = (int)($data['siteId'] ?? 0);
$pageId = (int)($data['pageId'] ?? 0);
$currentUserId = (int)$USER->GetID();

if ($siteId <= 0) {
    throw new RuntimeException('INVALID_SITE_ID');
}

if ($pageId <= 0) {
    throw new RuntimeException('INVALID_PAGE_ID');
}

/*
 * Смотреть список прав может только тот,
 * кто имеет право редактировать страницу
 * или глобально редактировать сайт.
 */
if (!PageAccessService::canEditPage($siteId, $pageId, $currentUserId)) {
    throw new RuntimeException('PAGE_ACCESS_DENIED');
}

$items = PageAccessRepository::listByPage($siteId, $pageId);

echo json_encode([
    'ok' => true,
    'data' => [
        'items' => $items,
    ],
], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);


---

3. Файл save.php

Файл:

/local/sitebuilder/api/actions/page_access/save.php

Код:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessRepository.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessService.php';

global $USER;

if (!is_object($USER) || !$USER->IsAuthorized()) {
    throw new RuntimeException('AUTH_REQUIRED');
}

if (!check_bitrix_sessid()) {
    throw new RuntimeException('BAD_SESSID');
}

$data = json_decode(file_get_contents('php://input'), true);

if (!is_array($data)) {
    $data = $_REQUEST;
}

$siteId = (int)($data['siteId'] ?? 0);
$pageId = (int)($data['pageId'] ?? 0);
$accessCode = (string)($data['accessCode'] ?? '');

$canView = !empty($data['canView']);
$canEdit = !empty($data['canEdit']);
$includeChildren = !empty($data['includeChildren']);

$currentUserId = (int)$USER->GetID();

if ($siteId <= 0) {
    throw new RuntimeException('INVALID_SITE_ID');
}

if ($pageId <= 0) {
    throw new RuntimeException('INVALID_PAGE_ID');
}

if ($accessCode === '') {
    throw new RuntimeException('EMPTY_ACCESS_CODE');
}

if (!$canView && !$canEdit) {
    throw new RuntimeException('EMPTY_PAGE_PERMISSION');
}

/*
 * Редактирование автоматически включает чтение.
 */
if ($canEdit) {
    $canView = true;
}

/*
 * Выдавать права может только тот,
 * кто сам может редактировать страницу.
 */
if (!PageAccessService::canEditPage($siteId, $pageId, $currentUserId)) {
    throw new RuntimeException('PAGE_ACCESS_DENIED');
}

$item = PageAccessRepository::save(
    $siteId,
    $pageId,
    $accessCode,
    $canView,
    $canEdit,
    $includeChildren,
    $currentUserId
);

echo json_encode([
    'ok' => true,
    'data' => [
        'item' => $item,
    ],
], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);


---

4. Файл delete.php

Файл:

/local/sitebuilder/api/actions/page_access/delete.php

Код:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessRepository.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessService.php';

global $USER;

if (!is_object($USER) || !$USER->IsAuthorized()) {
    throw new RuntimeException('AUTH_REQUIRED');
}

if (!check_bitrix_sessid()) {
    throw new RuntimeException('BAD_SESSID');
}

$data = json_decode(file_get_contents('php://input'), true);

if (!is_array($data)) {
    $data = $_REQUEST;
}

$id = (int)($data['id'] ?? 0);
$siteId = (int)($data['siteId'] ?? 0);
$pageId = (int)($data['pageId'] ?? 0);

$currentUserId = (int)$USER->GetID();

if ($id <= 0) {
    throw new RuntimeException('INVALID_PAGE_ACCESS_ID');
}

if ($siteId <= 0) {
    throw new RuntimeException('INVALID_SITE_ID');
}

if ($pageId <= 0) {
    throw new RuntimeException('INVALID_PAGE_ID');
}

/*
 * Удалять права может только тот,
 * кто может редактировать страницу.
 */
if (!PageAccessService::canEditPage($siteId, $pageId, $currentUserId)) {
    throw new RuntimeException('PAGE_ACCESS_DENIED');
}

PageAccessRepository::delete($id, $siteId, $pageId);

echo json_encode([
    'ok' => true,
    'data' => [
        'deleted' => true,
        'id' => $id,
    ],
], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);


---

5. Подключи actions в API router

Файл может быть один из этих:

/local/sitebuilder/api/index.php

или:

/local/sitebuilder/api/api.php

Найди там switch ($action) и добавь:

case 'pageAccess.list':
    require __DIR__ . '/actions/page_access/list.php';
    break;

case 'pageAccess.save':
    require __DIR__ . '/actions/page_access/save.php';
    break;

case 'pageAccess.delete':
    require __DIR__ . '/actions/page_access/delete.php';
    break;


---

6. Важно: router должен ловить ошибки и отдавать JSON

Если сейчас в API ещё нет нормального try/catch, сделай примерно так:

try {
    switch ($action) {
        case 'pageAccess.list':
            require __DIR__ . '/actions/page_access/list.php';
            break;

        case 'pageAccess.save':
            require __DIR__ . '/actions/page_access/save.php';
            break;

        case 'pageAccess.delete':
            require __DIR__ . '/actions/page_access/delete.php';
            break;

        default:
            throw new RuntimeException('UNKNOWN_ACTION');
    }
} catch (Throwable $e) {
    while (ob_get_level() > 0) {
        @ob_end_clean();
    }

    header('Content-Type: application/json; charset=UTF-8');

    echo json_encode([
        'ok' => false,
        'error' => 'SERVER_ERROR',
        'message' => $e->getMessage(),
        'details' => [
            'file' => $e->getFile(),
            'line' => $e->getLine(),
        ],
    ], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
}

Если у тебя уже есть общий обработчик ошибок — просто добавь новые case.


---

7. Проверка через консоль

Открой редактор сайта и вставь в консоль.

Выдать чтение пользователю

(async function () {
  const siteId = Number(prompt('siteId'));
  const pageId = Number(prompt('pageId'));
  const userId = Number(prompt('ID пользователя'));

  const res = await fetch('/local/sitebuilder/api/?action=pageAccess.save&sessid=' + BX.bitrix_sessid(), {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      siteId: siteId,
      pageId: pageId,
      accessCode: 'U' + userId,
      canView: true,
      canEdit: false,
      includeChildren: true
    })
  });

  console.log(await res.json());
})();

Выдать редактирование пользователю

(async function () {
  const siteId = Number(prompt('siteId'));
  const pageId = Number(prompt('pageId'));
  const userId = Number(prompt('ID пользователя'));

  const res = await fetch('/local/sitebuilder/api/?action=pageAccess.save&sessid=' + BX.bitrix_sessid(), {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      siteId: siteId,
      pageId: pageId,
      accessCode: 'U' + userId,
      canView: true,
      canEdit: true,
      includeChildren: true
    })
  });

  console.log(await res.json());
})();

Посмотреть права страницы

(async function () {
  const siteId = Number(prompt('siteId'));
  const pageId = Number(prompt('pageId'));

  const res = await fetch('/local/sitebuilder/api/?action=pageAccess.list', {
    method: 'POST',
    headers: {'Content-Type': 'application/json'},
    body: JSON.stringify({
      siteId: siteId,
      pageId: pageId
    })
  });

  console.log(await res.json());
})();


---

После этого у нас будет готов backend для выдачи прав. Следующий шаг — подключить проверки в API страниц: чтобы пользователь без page.edit не мог сохранять страницу и блоки.