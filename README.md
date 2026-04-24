Это нормально. SiteAccessSyncService.php — не страница и не endpoint, его нельзя открывать напрямую в браузере.

Ошибка:

{
  "ok": false,
  "error": "METHOD_NOT_ALLOWED"
}

означает, что ты либо открыл файл/endpoint через GET, либо попал не туда. Синхронизацию нужно запускать через основной API и только POST-запросом.

Правильный адрес:

/local/sitebuilder/api.php?action=site.syncAccess

POST-параметр:

siteId=11

Быстрая проверка через консоль браузера

Открой страницу портала, где ты авторизован, и в F12 → Console выполни:

const fd = new FormData();
fd.append('siteId', '11');

fetch('/local/sitebuilder/api.php?action=site.syncAccess', {
  method: 'POST',
  body: fd
})
  .then(r => r.json())
  .then(console.log)
  .catch(console.error);

Ожидаемый ответ должен быть примерно такой:

{
  "ok": true,
  "result": {
    "siteId": 11,
    "bitrixGroupId": 2,
    "created": 0,
    "updated": 0,
    "removed": 0,
    "kept": 1
  }
}

Что проверить, если не сработает

Если ответ будет:

NOT_MOVED_YET

значит блок site.syncAccess не вставлен в site.php или вставлен ниже финального sb_json_error('NOT_MOVED_YET'...).

Если будет:

SiteAccessSyncService.php не подключен

значит файл лежит не тут:

/srv/bx/docroot/local/sitebuilder/lib/SiteAccessSyncService.php

Сам файл сервиса напрямую не открываем. Работать должен только вызов через:

/local/sitebuilder/api.php?action=site.syncAccess