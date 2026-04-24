site.get уже работает, значит sessid и POST-формат нормальные.

Проблема сейчас конкретно такая:

{
  "ok": false,
  "error": "UNKNOWN_ACTION",
  "action": "site.syncAccess",
  "file": "/srv/bx/docroot/local/sitebuilder/api/index.php"
}

Это значит, что api/index.php не пропускает site.syncAccess в обработчик api/handlers/site.php.

1. Исправь /local/sitebuilder/api/index.php

Открой файл:

/local/sitebuilder/api/index.php

Найди там список action-ов для site.php. Скорее всего там что-то похожее:

$siteActions = [
    'site.list',
    'site.get',
    'site.create',
    'site.update',
    'site.delete',
    'site.setHome',
];

Добавь туда:

'site.syncAccess',

Должно стать примерно так:

$siteActions = [
    'site.list',
    'site.get',
    'site.create',
    'site.update',
    'site.delete',
    'site.setHome',
    'site.syncAccess',
];

Если у тебя там не массив, а if, например:

if (in_array($action, [
    'site.list',
    'site.get',
    ...
], true)) {
    require __DIR__ . '/handlers/site.php';
    exit;
}

то тоже просто добавь:

'site.syncAccess',


---

2. Проверь, что в site.php блок site.syncAccess стоит ДО финального NOT_MOVED_YET

В файле:

/local/sitebuilder/api/handlers/site.php

блок:

if ($action === 'site.syncAccess') {
    ...
}

должен быть выше этого куска:

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'site',
    'action' => $action,
    'file' => __FILE__,
]);

Если он ниже — он никогда не выполнится.


---

3. Важный момент: site.get сейчас не возвращает bitrixGroupId

В ответе site.get у тебя нет:

"bitrixGroupId": 2

Хотя при создании сайта он был.

Значит sb_find_site() или sb_read_sites() сейчас не мапит поле БД:

bitrix_group_id

в PHP-поле:

bitrixGroupId

После того как site.syncAccess начнет вызываться, может появиться ошибка:

EMPTY_BITRIX_GROUP_ID

Если появится — нужно будет поправить функцию нормализации сайта в storage_db. Но сначала добавь site.syncAccess в роутер api/index.php.


---

4. После правки проверь снова

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

Если после добавления action будет уже не UNKNOWN_ACTION, а другая ошибка — пришли её. Скорее всего следующим шагом нужно будет добавить bitrixGroupId в нормализацию сайта.