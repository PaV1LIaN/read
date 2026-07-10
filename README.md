Запрос page.list уже работает. Теперь нужно проверить, какие значения реально приходят у страниц, особенно navigationOnly, parentId и права доступа.

Запусти в консоли:

console.table(
  result.pages.map(page => ({
    id: page.id,
    title: page.title,
    slug: page.slug,
    parentId: page.parentId,
    navigationOnly: page.navigationOnly,
    access: JSON.stringify(page.access ?? null)
  }))
);

Если переменная result уже недоступна, выполни полный вариант:

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

  const result = await response.json();

  console.table(
    result.pages.map(page => ({
      id: page.id,
      title: page.title,
      slug: page.slug,
      parentId: page.parentId,
      navigationOnly: page.navigationOnly,
      access: JSON.stringify(page.access ?? null)
    }))
  );

  console.log('Полные данные страниц:', result.pages);

  return result;
})();

После этого проверь ожидаемую структуру:

parentId: 0 — страница корневая;

parentId: 14 — страница вложена в страницу «Диск»;

navigationOnly: true — пункт используется только как раздел навигации;

navigationOnly: false — обычная страница с содержимым;

access — индивидуальные настройки доступа страницы.


Сейчас обе страницы имеют parentId: 0, поэтому они обе находятся на верхнем уровне.

Следующий функциональный тест — получить блоки страницы «Тест» с ID 31:

(async function () {
  const fd = new FormData();

  fd.append('action', 'block.list');
  fd.append('sessid', BX.bitrix_sessid());
  fd.append('siteId', '13');
  fd.append('pageId', '31');

  const response = await fetch('/local/sitebuilder/api/index.php', {
    method: 'POST',
    body: fd,
    credentials: 'same-origin',
    cache: 'no-store'
  });

  const text = await response.text();

  console.log('Ответ сервера:', text);

  try {
    const result = JSON.parse(text);
    console.log('Блоки страницы:', result);
    return result;
  } catch (error) {
    console.error('API вернул не JSON');
    throw error;
  }
})();

Если block.list вернёт ok: true, значит цепочка работает полностью:

site → page.list → выбор страницы → block.list → отрисовка блоков

Тогда дальше уже нужно исправлять отображение дерева страниц в editor.php, а не API.