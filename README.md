GET 405 — это нормально, потому что API принимает только POST.

Сейчас важно увидеть тело ответа 403, а не выполнять console.log('TEXT:', text) отдельно. Переменная text существует только внутри .then.

Выполни в консоли вот так целиком:

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

Если в консоли будет:

SESSID: ""

значит запрос уходит без сессии, поэтому и 403.

Тогда временно проверь так:

document.querySelector('.sb-disk')?.getAttribute('data-sessid')

Если там есть значение, его можно использовать.

Если после выполнения кода будет STATUS: 403, пришли именно строку:

TEXT: ...

Там будет понятно, это BAD_SESSID, ACCESS_DENIED, OWNER_REQUIRED или другая причина.