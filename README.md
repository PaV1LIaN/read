site.get тоже даёт 400, значит проблема сейчас не в site.syncAccess, а в формате тестового запроса или в общей проверке API.

Скорее всего твой /local/sitebuilder/api.php принимает данные как JSON, потому что фронт у нас почти везде дергает API через Content-Type: application/json.

Попробуй так:

fetch('/local/sitebuilder/api.php?action=site.get', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    siteId: 11,
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

Если site.get так заработает, тогда синхронизацию запускай так:

fetch('/local/sitebuilder/api.php?action=site.syncAccess', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    siteId: 11,
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

Если и JSON-запрос даст 400, открой в DevTools → Network → выбери запрос site.get или site.syncAccess → вкладка Response и скопируй сюда тело ответа. Там будет конкретная ошибка, например BAD_SESSID, SITE_ID_REQUIRED, ACCESS_DENIED или что-то из bootstrap API.