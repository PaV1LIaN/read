(async function () {
  try {
    const sessidResponse = await fetch(
      '/local/sitebuilder/api/get_sessid.php',
      {
        method: 'GET',
        credentials: 'same-origin',
        cache: 'no-store'
      }
    );

    const sessidText = await sessidResponse.text();

    console.log('1. HTTP get_sessid:', sessidResponse.status);
    console.log('2. Ответ get_sessid:', sessidText);

    let sessidData;

    try {
      sessidData = JSON.parse(sessidText);
    } catch (error) {
      console.error('get_sessid.php вернул не JSON');
      return;
    }

    if (!sessidData.ok || !sessidData.sessid) {
      console.error('Не удалось получить sessid:', sessidData);
      return;
    }

    const siteId = Number(prompt('Введите siteId'));
    const pageId = Number(prompt('Введите pageId'));
    const userId = Number(prompt('Введите ID пользователя'));

    if (siteId <= 0 || pageId <= 0 || userId <= 0) {
      console.error('siteId, pageId и userId должны быть больше нуля');
      return;
    }

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

    const responseText = await response.text();

    console.log('3. HTTP pageAccess.save:', response.status);
    console.log('4. Ответ pageAccess.save:', responseText);

    try {
      const result = JSON.parse(responseText);
      console.log('5. JSON:', result);

      if (result.ok) {
        console.log('УСПЕШНО: право сохранено');
      } else {
        console.error('ОШИБКА API:', result);
      }
    } catch (error) {
      console.error('pageAccess.save вернул не JSON');
    }
  } catch (error) {
    console.error('Ошибка выполнения запроса:', error);
  }
})();
Promise {<pending>}
VM239:51  POST https://portal24.itsnn.ru/local/sitebuilder/api/index.php 400 (Bad Request)
(anonymous) @ VM239:51
await in (anonymous)
(anonymous) @ VM239:81
VM239:73 ОШИБКА API: {ok: false, error: 'SQLSTATE[42P10]: Invalid column reference: 7 ОШИБК…исключения, соответствующего указанию ON CONFLICT', action: 'pageAccess.save', file: '/srv/bx/docroot/local/sitebuilder/lib/PageAccessRepository.php', line: 132}
