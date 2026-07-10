(async function () {
  const sessidResponse = await fetch(
    '/local/sitebuilder/api/get_sessid.php',
    {
      credentials: 'same-origin',
      cache: 'no-store'
    }
  );

  const sessidData = await sessidResponse.json();

  if (!sessidData.ok || !sessidData.sessid) {
    throw new Error('Не удалось получить sessid');
  }

  const siteId = Number(prompt('Введите siteId'));
  const pageId = Number(prompt('Введите pageId'));
  const userId = Number(prompt('Введите ID пользователя'));

  const fd = new FormData();

  fd.append('action', 'pageAccess.save');
  fd.append('sessid', sessidData.sessid);
  fd.append('siteId', String(siteId));
  fd.append('pageId', String(pageId));
  fd.append('accessCode', 'U' + userId);
  fd.append('canView', '1');
  fd.append('canEdit', '1');
  fd.append('includeChildren', '1');

  const response = await fetch(
    '/local/sitebuilder/api/index.php',
    {
      method: 'POST',
      body: fd,
      credentials: 'same-origin',
      cache: 'no-store'
    }
  );

  const text = await response.text();

  let data;

  try {
    data = JSON.parse(text);
  } catch (error) {
    data = {
      ok: false,
      error: 'Ответ сервера не является JSON',
      raw: text
    };
  }

  const result = {
    httpStatus: response.status,
    ...data
  };

  console.log('ИТОГ ПРОВЕРКИ:', result);

  return result;
})();
Promise {<pending>}
[[Prototype]]
: 
Promise
[[PromiseState]]
: 
"fulfilled"
[[PromiseResult]]
: 
Object
