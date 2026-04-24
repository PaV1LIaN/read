403 сейчас почти точно из-за пустого sessid.

Ты проверил:

document.querySelector('.sb-disk')?.getAttribute('data-sessid')

и получил:

undefined

Значит ты выполняешь проверку не на странице сайта/редактора, где есть компонент и Bitrix-сессия, а на странице API или другой странице без BX.

Как правильно проверить

Открой не API-ссылку, а страницу sitebuilder, например:

/local/sitebuilder/index.php

или страницу редактора/админки, где ты создаешь сайты.

Там в консоли сначала проверь:

typeof BX !== 'undefined'

Потом:

typeof BX !== 'undefined' && typeof BX.bitrix_sessid === 'function' ? BX.bitrix_sessid() : 'NO_SESSID'

Если вернулся нормальный хеш, тогда запускай:

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

Почему GET дает 405

Это нормально:

GET /local/sitebuilder/api.php?action=site.syncAccess → 405

Твой API принимает только POST.

Если на странице sitebuilder BX тоже нет

Тогда нужно передавать sessid прямо из PHP в страницу. В index.php или редакторе, где ты делаешь запросы к API, должен быть доступен:

<?= bitrix_sessid() ?>

И JS должен отправлять:

fd.append('sessid', window.BX ? BX.bitrix_sessid() : '...');

Но сначала проверь именно с нормальной страницы sitebuilder, не с URL api.php?action=site.syncAccess.