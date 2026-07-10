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
Promise {<rejected>: ReferenceError: BX is not defined
    at <anonymous>:9:23
    at <anonymous>:23:3}
[[Prototype]]
: 
Promise
[[PromiseState]]
: 
"rejected"
[[PromiseResult]]
: 
ReferenceError: BX is not defined at <anonymous>:9:23 at <anonymous>:23:3
