Отлично, индекс создан правильно:

uq_page_access_site_page_code
(site_id, page_id, access_code)

Теперь ошибка ON CONFLICT устранена.

Результат:

PromiseState: fulfilled
PromiseResult: undefined

сам по себе не показывает ответ API — функция просто ничего не возвращает. Нужно повторно выполнить тест после создания индекса.

Вставь этот сокращённый вариант. Он вернёт итог прямо в результат Promise:

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

Теперь при успехе в консоли будет:

ИТОГ ПРОВЕРКИ:
{
    httpStatus: 200,
    ok: true,
    data: {
        item: ...
    }
}

Существующий индекс:

page_access_page_id_access_code_key

пока не удаляй. Он дублирует часть новой проверки, но работе не мешает. После успешного теста можно удалить временный файл:

/local/sitebuilder/migrations/fix_page_access_unique.php