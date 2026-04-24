Причина понятная: твой /local/sitebuilder/api/index.php берёт action не из URL, а из $_POST.

Поэтому запрос вида:

/local/sitebuilder/api.php?action=site.syncAccess

для твоего API приходит как:

"action": ""

Нужно передавать action в POST body.

Проверь site.get вот так

fetch('/local/sitebuilder/api.php', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    action: 'site.get',
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

Потом site.syncAccess

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

Ожидаемо должно быть примерно так:

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

Что поправить потом в JS API helper

Если у тебя где-то фронт вызывает так:

fetch('/local/sitebuilder/api.php?action=' + action, ...)

лучше привести к единому виду:

const body = new URLSearchParams();
body.append('action', action);
body.append('sessid', BX.bitrix_sessid());

И отправлять action именно в POST. Сейчас твой API так и ожидает.