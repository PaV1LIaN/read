Promise {<fulfilled>: undefined} здесь не ошибка. Это значит, что асинхронная функция завершилась без return.

И второй результат:

undefined

тоже нормальный: console.log() сам возвращает undefined.

Но нам нужно увидеть значения внутри объекта, а не результат console.log.

Вставь в консоль только это выражение, без console.log:

({
  BX: typeof window.BX,
  hiddenSessid: document.querySelector('input[name="sessid"]')?.value ?? null,
  dataSessid: document.querySelector('[data-sessid]')?.getAttribute('data-sessid') ?? null,
  diskSessid: document.querySelector('.sb-disk')?.__diskComponent?.getSessid?.() ?? null,
  cookie: document.cookie
})

Консоль должна показать объект примерно так:

{
  BX: "undefined",
  hiddenSessid: null,
  dataSessid: null,
  diskSessid: null,
  cookie: "..."
}

Судя по текущему поведению, предыдущий скрипт, вероятнее всего, остановился здесь:

if (!sessid) {
  console.error('sessid не найден на странице');
  return;
}

То есть запрос к API вообще не отправлялся.

Надёжная проверка через отдельный endpoint

Создай временный файл:

/local/sitebuilder/api/get_sessid.php

Полный код:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

global $USER;

header('Content-Type: application/json; charset=UTF-8');

if (!is_object($USER) || !$USER->IsAuthorized()) {
    http_response_code(401);

    echo json_encode([
        'ok' => false,
        'error' => 'AUTH_REQUIRED',
    ], JSON_UNESCAPED_UNICODE);

    exit;
}

echo json_encode([
    'ok' => true,
    'sessid' => bitrix_sessid(),
    'userId' => (int)$USER->GetID(),
], JSON_UNESCAPED_UNICODE);

Теперь вставь в консоль:

(async function () {
  try {
    const sessidResponse = await fetch(
      '/local/sitebuilder/api/get_sessid.php',
      {
        method: 'GET',
        credentials: 'same-origin',
        cache: 'no-store'
      }
    );

    const sessidText = await sessidResponse.text();

    console.log('Ответ get_sessid.php:', sessidText);

    const sessidData = JSON.parse(sessidText);

    if (!sessidData.ok || !sessidData.sessid) {
      console.error('Не удалось получить sessid:', sessidData);
      return;
    }

    const siteId = Number(prompt('siteId'));
    const pageId = Number(prompt('pageId'));
    const userId = Number(prompt('ID пользователя, кому выдать доступ'));

    if (!siteId || !pageId || !userId) {
      console.error('siteId, pageId и userId должны быть больше нуля');
      return;
    }

    const fd = new FormData();

    fd.append('action', 'pageAccess.save');
    fd.append('sessid', sessidData.sessid);
    fd.append('siteId', String(siteId));
    fd.append('pageId', String(pageId));
    fd.append('accessCode', 'U' + userId);
    fd.append('canView', '1');
    fd.append('canEdit', '1');
    fd.append('includeChildren', '1');

    const response = await fetch(
      '/local/sitebuilder/api/index.php',
      {
        method: 'POST',
        body: fd,
        credentials: 'same-origin'
      }
    );

    const responseText = await response.text();

    console.log('HTTP:', response.status);
    console.log('Ответ pageAccess.save:', responseText);

    try {
      console.log('JSON:', JSON.parse(responseText));
    } catch (error) {
      console.error('Сервер вернул не JSON:', error);
    }
  } catch (error) {
    console.error('Ошибка выполнения:', error);
  }
})();

Ожидаемый успешный результат:

HTTP: 200

и:

{
  "ok": true,
  "data": {
    "item": {
      "id": 1,
      "siteId": 1,
      "pageId": 1,
      "accessCode": "U99",
      "canView": true,
      "canEdit": true,
      "includeChildren": true
    }
  }
}

После проверки файл:

/local/sitebuilder/api/get_sessid.php

лучше удалить. Постоянно sessid позже добавим прямо в конфигурацию редактора, чтобы не создавать отдельный запрос.