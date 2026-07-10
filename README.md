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

    console.log('Ответ get_sessid.php:', sessidText);

    const sessidData = JSON.parse(sessidText);

    if (!sessidData.ok || !sessidData.sessid) {
      console.error('Не удалось получить sessid:', sessidData);
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
        credentials: 'same-origin'
      }
    );

    const responseText = await response.text();

    console.log('HTTP:', response.status);
    console.log('Ответ pageAccess.save:', responseText);

    try {
      console.log('JSON:', JSON.parse(responseText));
    } catch (error) {
      console.error('Сервер вернул не JSON:', error);
    }
  } catch (error) {
    console.error('Ошибка выполнения:', error);
  }
})();
