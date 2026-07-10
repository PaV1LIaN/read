(async function () {
  const diskComponent = document.querySelector('.sb-disk')?.__diskComponent;

  const sessid =
    window.BX?.bitrix_sessid?.() ||
    diskComponent?.getSessid?.() ||
    document.querySelector('input[name="sessid"]')?.value ||
    document.querySelector('[data-sessid]')?.getAttribute('data-sessid') ||
    '';

  if (!sessid) {
    console.error('sessid не найден на странице');
    console.log(
      'Элементы с sessid:',
      Array.from(document.querySelectorAll('[name="sessid"], [data-sessid]'))
    );
    return;
  }

  const siteId = Number(prompt('siteId'));
  const pageId = Number(prompt('pageId'));
  const userId = Number(prompt('ID пользователя, кому выдать доступ'));

  if (!siteId || !pageId || !userId) {
    console.error('siteId, pageId и userId должны быть больше нуля');
    return;
  }

  const fd = new FormData();

  fd.append('action', 'pageAccess.save');
  fd.append('sessid', sessid);
  fd.append('siteId', String(siteId));
  fd.append('pageId', String(pageId));
  fd.append('accessCode', 'U' + userId);
  fd.append('canView', '1');
  fd.append('canEdit', '1');
  fd.append('includeChildren', '1');

  const response = await fetch('/local/sitebuilder/api/index.php', {
    method: 'POST',
    body: fd,
    credentials: 'same-origin'
  });

  const text = await response.text();

  console.log('HTTP:', response.status);
  console.log('Ответ:', text);

  try {
    console.log('JSON:', JSON.parse(text));
  } catch (error) {
    console.error('Сервер вернул не JSON');
  }
})();
Promise {<fulfilled>: undefined}[[Prototype]]: Promise[[PromiseState]]: "fulfilled"[[PromiseResult]]: undefined
console.log({
  BX: typeof window.BX,
  hiddenSessid: document.querySelector('input[name="sessid"]')?.value,
  dataSessid: document.querySelector('[data-sessid]')?.getAttribute('data-sessid'),
  diskSessid: document.querySelector('.sb-disk')?.__diskComponent?.getSessid?.()
});
undefined
