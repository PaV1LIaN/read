Всё работает корректно.

Ответ API:

ok: true — запрос успешен;

PromiseState: fulfilled — Promise завершён;

globalEdit: true — есть право редактирования;

globalView: true — есть право просмотра;

hasPageAccess: true — доступ к страницам разрешён;

API вернул 2 страницы:

ID 14 — «Диск»;

ID 31 — «Тест».



Promise {<pending>} — не ошибка. Консоль сначала показывает Promise сразу после запуска, а после выполнения его состояние меняется на fulfilled.

Чтобы консоль сразу вывела итоговый объект без отдельного Promise, можно запустить так:

const result = await (async function () {
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

  return await response.json();
})();

console.table(result.pages);
result;

Ожидаемый вывод таблицы:

id   title   slug   parentId
14   Диск    disk   0
31   Тест    test   0

Значит проблема с получением sessid и вызовом page.list устранена.