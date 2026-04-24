400 Bad Request уже значит, что sessid прошёл дальше, но сам API вернул ошибку по данным/логике.

Сейчас нужно увидеть тело ответа, там будет точная причина. В консоли после выполнения должны появиться строки:

STATUS:
TEXT:
JSON:

Ты прислал только строку Network про 400, а не сам TEXT.

Выполни вот так и дождись вывода:

fetch('/local/sitebuilder/api.php?action=site.syncAccess', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
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

Если снова будет 400, скопируй сюда именно строку:

TEXT: ...

Ещё можно параллельно проверить, видит ли API сайт №11:

fetch('/local/sitebuilder/api.php?action=site.get', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    siteId: '11',
    sessid: BX.bitrix_sessid()
  }),
  credentials: 'same-origin'
})
.then(r => r.text())
.then(console.log);

Если site.get работает, а site.syncAccess даёт 400, значит проблема уже внутри нового action или SiteAccessSyncService.