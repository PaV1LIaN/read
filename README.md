api.php?action=site.syncAccess:1  GET https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess 405 (Method Not Allowed)
(async function () {
  const sessid =
    (typeof BX !== 'undefined' && typeof BX.bitrix_sessid === 'function' && BX.bitrix_sessid()) ||
    document.querySelector('input[name="sessid"]')?.value ||
    document.querySelector('.sb-disk')?.getAttribute('data-sessid') ||
    '';

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
Promise {<pending>}[[Prototype]]: Promisecatch: ƒ catch()length: 1name: "catch"arguments: (...)caller: (...)[[Prototype]]: ƒ ()[[Scopes]]: Scopes[0]constructor: ƒ Promise()all: ƒ all()allSettled: ƒ allSettled()any: ƒ any()length: 1name: "Promise"prototype: Promise {Symbol(Symbol.toStringTag): 'Promise', then: ƒ, catch: ƒ, finally: ƒ}race: ƒ race()reject: ƒ reject()resolve: ƒ resolve()try: ƒ try()withResolvers: ƒ withResolvers()Symbol(Symbol.species): ƒ Promise()Symbol(Symbol.species): ƒ Promise()arguments: (...)caller: (...)[[Prototype]]: ƒ ()[[Scopes]]: Scopes[0]finally: ƒ finally()length: 1name: "finally"arguments: (...)caller: (...)[[Prototype]]: ƒ ()[[Scopes]]: Scopes[0]then: ƒ then()length: 2name: "then"arguments: (...)caller: (...)[[Prototype]]: ƒ ()[[Scopes]]: Scopes[0]Symbol(Symbol.toStringTag): "Promise"[[Prototype]]: Object[[PromiseState]]: "fulfilled"[[PromiseResult]]: undefined
VM44:14  POST https://portal24.itsnn.ru/local/sitebuilder/api.php?action=site.syncAccess 403 (Forbidden)
(anonymous) @ VM44:14
(anonymous) @ VM44:30
document.querySelector('.sb-disk')?.getAttribute('data-sessid')
undefined
