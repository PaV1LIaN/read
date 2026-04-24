GET по ссылке в браузере — это нормально, там будет:

{
  "ok": false,
  "error": "METHOD_NOT_ALLOWED"
}

А вот POST 403 почти наверняка из-за того, что в запросе нет sessid.

Попробуй так в консоли:

const fd = new FormData();

fd.append('siteId', '11');

fd.append(
  'sessid',
  typeof BX !== 'undefined' && typeof BX.bitrix_sessid === 'function'
    ? BX.bitrix_sessid()
    : ''
);

fetch('/local/sitebuilder/api.php?action=site.syncAccess', {
  method: 'POST',
  body: fd,
  credentials: 'same-origin'
})
  .then(async r => {
    const text = await r.text();
    console.log('STATUS:', r.status);
    console.log('TEXT:', text);

    try {
      console.log('JSON:', JSON.parse(text));
    } catch (e) {}
  })
  .catch(console.error);

Если всё нормально, должен быть ответ примерно:

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

Если снова будет 403, пришли именно то, что выведет:

console.log('TEXT:', text);

Потому что там будет понятно, это BAD_SESSID, ACCESS_DENIED, OWNER_REQUIRED или другая ошибка.