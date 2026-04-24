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
Promise {<pending>}
VM10651:1  POST https://portal24.itsnn.ru/local/sitebuilder/api.php 500 (Internal Server Error)

{
    "ok": false,
    "error": "EMPTY_BITRIX_GROUP_ID",
    "handler": "site",
    "action": "site.syncAccess",
    "file": "\/srv\/bx\/docroot\/local\/sitebuilder\/api\/handlers\/site.php"
}
