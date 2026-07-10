BX is not defined означает, что на этой странице не подключено JavaScript-ядро Битрикса. Сам PHP-Битрикс работает, но функции BX.bitrix_sessid() в браузере нет.

Используй код, который ищет sessid несколькими способами, включая компонент диска и скрытые поля:

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

Если код напишет:

sessid не найден на странице

вставь отдельно эту диагностику:

console.log({
  BX: typeof window.BX,
  hiddenSessid: document.querySelector('input[name="sessid"]')?.value,
  dataSessid: document.querySelector('[data-sessid]')?.getAttribute('data-sessid'),
  diskSessid: document.querySelector('.sb-disk')?.__diskComponent?.getSessid?.()
});

Нормальное постоянное решение — передавать bitrix_sessid() в конфигурацию редактора при генерации страницы, чтобы все API-запросы использовали один готовый sessid, даже когда BX не подключён.