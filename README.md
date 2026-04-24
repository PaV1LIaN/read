fetch('/local/sitebuilder/api.php?action=site.get&sessid=' + encodeURIComponent(BX.bitrix_sessid()), {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    siteId: 11
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
VM10380:1  POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.get&sessid=01b508fb9a23c3b207d3d7de3254634c 400 (Bad Request)
(anonymous) @ VM10380:1
fetch('/local/sitebuilder/api.php?action=site.syncAccess&sessid=' + encodeURIComponent(BX.bitrix_sessid()), {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    siteId: 11
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
VM10384:1  POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess&sessid=01b508fb9a23c3b207d3d7de3254634c 400 (Bad Request)
