OK: migration completed through /local/sitebuilder/lib/db.php

Array
(
    [page_access] => page_access
    [page] => 
    [access] => access
)

(async function () {
  const siteId = Number(prompt('siteId'));
  const pageId = Number(prompt('pageId'));
  const userId = Number(prompt('ID пользователя, кому выдать доступ'));

  const fd = new FormData();

  fd.append('action', 'pageAccess.save');
  fd.append('sessid', BX.bitrix_sessid());
  fd.append('siteId', siteId);
  fd.append('pageId', pageId);
  fd.append('accessCode', 'U' + userId);
  fd.append('canView', '1');
  fd.append('canEdit', '1');
  fd.append('includeChildren', '1');

  const res = await fetch('/local/sitebuilder/api/index.php', {
    method: 'POST',
    body: fd
  });

  console.log(await res.json());
})();

Если всё нормально, должен вернуться:

{
  "ok": true,
  "data": {
    "item": {
      "siteId": 1,
      "pageId": 1,
      "accessCode": "U99",
      "canView": true,
      "canEdit": true,
      "includeChildren": true
    }
  }
}
VM13:25 Uncaught SyntaxError: Unexpected identifier 'всё'
