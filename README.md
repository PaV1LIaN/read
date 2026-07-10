Теперь запрос отработал успешно:

PromiseState: fulfilled — запрос завершён;

ok: true — API не вернул ошибку;

найдено две страницы: «Диск» и «Тест»;

доступ есть: globalEdit, globalView, hasPageAccess равны true.


Важный момент: pages находится прямо в result, а не в result.data. Поэтому выводить нужно так:

console.table(
  result.pages?.map(page => ({
    id: page.id,
    title: page.title,
    slug: page.slug,
    parentId: page.parentId,
    navigationOnly: page.navigationOnly,
    access: page.access
  })) ?? []
);

Полный исправленный вариант:

(async function () {
  const fd = new FormData();

  fd.append('action', 'page.list');
  fd.append('sessid', BX.bitrix_sessid());
  fd.append('siteId', '13');

  const response = await fetch('/local/sitebuilder/api/index.php', {
    method: 'POST',
    body: fd,
    credentials: 'same-origin',
    cache: 'no-store'
  });

  const text = await response.text();

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${text}`);
  }

  const result = JSON.parse(text);

  console.log('Полный ответ:', result);

  console.table(
    result.pages?.map(page => ({
      id: page.id,
      title: page.title,
      slug: page.slug,
      parentId: page.parentId,
      navigationOnly: page.navigationOnly,
      access: page.access
    })) ?? []
  );

  return result;
})();

Promise {<pending>} при запуске — нормальное поведение консоли. Затем он перешёл в состояние fulfilled, то есть всё выполнено без ошибки.