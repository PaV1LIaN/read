Ошибка состоит из двух частей:

1. /local/sitebuilder/api/get_sessid.php не существует, поэтому сервер возвращает 404 File not found.


2. Код пытается разобрать текст File not found как JSON, поэтому появляется Unexpected token 'F'.



На странице Битрикс отдельный get_sessid.php обычно не нужен. Возьми sessid через BX.bitrix_sessid():

(async function () {
  if (typeof BX === 'undefined' || typeof BX.bitrix_sessid !== 'function') {
    throw new Error('BX.bitrix_sessid() недоступен. Запусти код на странице Битрикс24.');
  }

  const fd = new FormData();

  fd.append('action', 'page.list');
  fd.append('sessid', BX.bitrix_sessid());
  fd.append('siteId', '13');

  const response = await fetch('/local/sitebuilder/api/index.php', {
    method: 'POST',
    body: fd,
    credentials: 'same-origin',
    cache: 'no-store'
  });

  const responseText = await response.text();

  if (!response.ok) {
    throw new Error(
      `HTTP ${response.status}: ${responseText}`
    );
  }

  let result;

  try {
    result = JSON.parse(responseText);
  } catch (error) {
    console.error('Сервер вернул не JSON:', responseText);
    throw error;
  }

  console.table(
    result.data?.pages?.map(page => ({
      id: page.id,
      title: page.title,
      parentId: page.parentId,
      navigationOnly: page.navigationOnly,
      access: page.access
    })) ?? []
  );

  return result;
})();

Promise {<pending>} вначале — это нормально: асинхронная функция ещё выполнялась. Основная проблема именно в 404.

Если тебе всё же нужен отдельный файл, создай:

/local/sitebuilder/api/get_sessid.php

<?php

declare(strict_types=1);

define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('NOT_CHECK_PERMISSIONS', true);

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Context;

global $USER;

header('Content-Type: application/json; charset=UTF-8');

if (!$USER || !$USER->IsAuthorized()) {
    http_response_code(401);

    echo json_encode(
        [
            'success' => false,
            'error' => 'AUTH_REQUIRED',
        ],
        JSON_UNESCAPED_UNICODE
    );

    exit;
}

echo json_encode(
    [
        'success' => true,
        'sessid' => bitrix_sessid(),
    ],
    JSON_UNESCAPED_UNICODE
);

Но для проверки из консоли браузера лучше использовать первый вариант с BX.bitrix_sessid().