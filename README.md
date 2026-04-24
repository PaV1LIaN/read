GET https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess 405 (Method Not Allowed)
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
Promise {<pending>}
VM30:12  POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess 403 (Forbidden)
(anonymous) @ VM30:12
console.log('TEXT:', text);
VM34:1 Uncaught ReferenceError: text is not defined
    at <anonymous>:1:22
