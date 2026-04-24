typeof BX !== 'undefined'
true
typeof BX !== 'undefined' && typeof BX.bitrix_sessid === 'function' ? BX.bitrix_sessid() : 'NO_SESSID'
'01b508fb9a23c3b207d3d7de3254634c'
(async function () {
  const sessid = BX.bitrix_sessid();

  console.log('SESSID:', sessid);

  const fd = new FormData();
  fd.append('siteId', '11');
  fd.append('sessid', sessid);

  const r = await fetch('/local/sitebuilder/api.php?action=site.syncAccess', {
    method: 'POST',
    body: fd,
    credentials: 'same-origin'
  });

  const text = await r.text();

  console.log('STATUS:', r.status);
  console.log('TEXT:', text);

  try {
    console.log('JSON:', JSON.parse(text));
  } catch (e) {
    console.log('NOT JSON');
  }
})();
Promise {<pending>}
VM496:10  POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess 400 (Bad Request)
