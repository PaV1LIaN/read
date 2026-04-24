Сейчас 400 потому что ты отправляешь siteId в JSON, а твой site.php читает так:

$siteId = (int)($_POST['siteId'] ?? 0);

То есть siteId должен попасть именно в $_POST, а не в JSON body.

Проверь так:

fetch('/local/sitebuilder/api.php?action=site.get&sessid=' + encodeURIComponent(BX.bitrix_sessid()), {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    siteId: '11'
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

Потом синхронизацию так:

fetch('/local/sitebuilder/api.php?action=site.syncAccess&sessid=' + encodeURIComponent(BX.bitrix_sessid()), {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
  },
  body: new URLSearchParams({
    siteId: '11'
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

Если site.get сработает, а site.syncAccess нет — значит проблема уже внутри нового action.

Если снова будет 400, открой DevTools → Network → запрос site.get или site.syncAccess → вкладка Response и скопируй тело ответа. Сейчас без текста ответа видно только статус, но не причину.