BAD_SESSID означает: запрос до API дошёл, но api.php не увидел sessid там, где он его проверяет.

Сейчас попробуй самый надёжный вариант: передать sessid прямо в URL, а данные оставить JSON.

1. Проверь site.get

Выполни в консоли:

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

Если всё нормально, должен вернуться сайт.


---

2. Потом проверь site.syncAccess

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


---

Почему так

Сейчас ты отправляешь:

body: JSON.stringify({
  siteId: 11,
  sessid: BX.bitrix_sessid()
})

Но если в api.php проверка check_bitrix_sessid() выполняется до разбора JSON, Bitrix не видит sessid внутри JSON-тела.

А если передать так:

/local/sitebuilder/api.php?action=site.get&sessid=...

то sessid попадает в $_REQUEST, и check_bitrix_sessid() должен его увидеть.


---

Если это сработает

Тогда надо поправить общий JS API helper в sitebuilder, чтобы он всегда добавлял sessid в URL:

const url = '/local/sitebuilder/api.php?action=' + encodeURIComponent(action)
  + '&sessid=' + encodeURIComponent(BX.bitrix_sessid());

А из тела sessid можно убрать.


---

Если и с &sessid=... будет BAD_SESSID, пришли начало файла:

/local/sitebuilder/api.php

и файл:

/local/sitebuilder/api/bootstrap.php

Тогда поправим саму проверку sessid в API.