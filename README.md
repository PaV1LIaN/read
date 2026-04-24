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
Promise {<pending>}
[[Prototype]]
: 
Promise
[[PromiseState]]
: 
"fulfilled"
[[PromiseResult]]
: 
undefined
VM582:1 
 POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess 400 (Bad Request)
(anonymous)	@	VM582:1
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
Promise {<pending>}
[[Prototype]]
: 
Promise
[[PromiseState]]
: 
"fulfilled"
[[PromiseResult]]
: 
undefined
VM586:1 
 POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.get 400 (Bad Request)
(anonymous)	@	VM586:1
