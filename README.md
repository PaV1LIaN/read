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
Promise {<pending>}
[[Prototype]]
: 
Promise
[[PromiseState]]
: 
"fulfilled"
[[PromiseResult]]
: 
undefined



OK: unique index created

Array
(
    [0] => Array
        (
            [indexname] => ix_page_access_access_code
            [indexdef] => CREATE INDEX ix_page_access_access_code ON sitebuilder.page_access USING btree (access_code)
        )

    [1] => Array
        (
            [indexname] => ix_page_access_page_id
            [indexdef] => CREATE INDEX ix_page_access_page_id ON sitebuilder.page_access USING btree (page_id)
        )

    [2] => Array
        (
            [indexname] => ix_page_access_site_id
            [indexdef] => CREATE INDEX ix_page_access_site_id ON sitebuilder.page_access USING btree (site_id)
        )

    [3] => Array
        (
            [indexname] => page_access_page_id_access_code_key
            [indexdef] => CREATE UNIQUE INDEX page_access_page_id_access_code_key ON sitebuilder.page_access USING btree (page_id, access_code)
        )

    [4] => Array
        (
            [indexname] => page_access_pkey
            [indexdef] => CREATE UNIQUE INDEX page_access_pkey ON sitebuilder.page_access USING btree (id)
        )

    [5] => Array
        (
            [indexname] => uq_page_access_site_page_code
            [indexdef] => CREATE UNIQUE INDEX uq_page_access_site_page_code ON sitebuilder.page_access USING btree (site_id, page_id, access_code)
        )

)
