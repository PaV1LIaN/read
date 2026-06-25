Вставь это в консоль браузера на странице, где открыт компонент Диск.

Код загрузит тестовый файл указанного размера, покажет кто режет, а если загрузка пройдёт — попробует удалить тестовый файл обратно.

(async function () {
  const root = document.querySelector('.sb-disk');
  const comp = root && root.__diskComponent;

  if (!root || !comp) {
    console.error('Компонент Диск не найден. Открой страницу с диском и обнови её.');
    return;
  }

  const mbText = prompt('Размер тестового файла в МБ', '60');
  const mb = Number(String(mbText || '').replace(',', '.'));

  if (!Number.isFinite(mb) || mb <= 0) {
    console.error('Неверный размер');
    return;
  }

  const bytes = Math.round(mb * 1024 * 1024);
  const testFileName = '__sb_upload_limit_test_' + Date.now() + '_' + mb + 'MB.tmp';

  console.group('SiteBuilder Disk upload limit diagnostic');

  console.log('Тестовый размер:', mb + ' МБ', '(' + bytes + ' байт)');
  console.log('Текущая папка:', comp.state.currentFolderId);
  console.log('rootFolderId:', comp.state.rootFolderId);
  console.log('siteId/pageId/blockId:', comp.state.siteId, comp.state.pageId, comp.state.blockId);

  const settings = comp.state.settings || {};
  const maxFileSize = Number(settings.maxFileSize || 0);

  console.log('Настройка компонента maxFileSize:', maxFileSize, maxFileSize ? '(' + Math.round(maxFileSize / 1024 / 1024) + ' МБ)' : '(не задано)');

  if (maxFileSize > 0 && bytes > maxFileSize) {
    console.warn(
      'Скорее всего блокирует НАШ КОМПОНЕНТ DISK: размер файла больше maxFileSize. ' +
      'Подними maxFileSize в настройках компонента.'
    );
  } else {
    console.log('По настройке maxFileSize компонент НЕ должен блокировать этот размер.');
  }

  const formData = new FormData();

  formData.append('siteId', comp.state.siteId);
  formData.append('pageId', comp.state.pageId);
  formData.append('blockId', comp.state.blockId);
  formData.append('currentFolderId', comp.state.currentFolderId || comp.state.rootFolderId);
  formData.append('sessid', comp.getSessid());

  const blob = new Blob([new Uint8Array(bytes)], {
    type: 'application/octet-stream'
  });

  formData.append('files[]', blob, testFileName);

  const url = '/local/sitebuilder/components/disk/api.php?action=upload';

  console.log('Начинаю тестовую загрузку:', testFileName);

  const startedAt = Date.now();

  let response;
  let text = '';

  try {
    response = await fetch(url, {
      method: 'POST',
      body: formData,
      credentials: 'same-origin'
    });

    text = await response.text();
  } catch (e) {
    console.error('Сетевая ошибка fetch:', e);
    console.warn('Возможные причины: соединение оборвалось, прокси/Angie закрыл запрос, CORS/авторизация.');
    console.groupEnd();
    return;
  }

  const seconds = ((Date.now() - startedAt) / 1000).toFixed(1);

  console.log('HTTP status:', response.status, response.statusText);
  console.log('Время запроса:', seconds + ' сек.');

  let json = null;

  try {
    json = JSON.parse(text);
  } catch (e) {
    json = null;
  }

  if (response.status === 413) {
    console.error('БЛОКИРУЕТ ANGIE/NGINX: 413 Request Entity Too Large');
    console.warn('Нужно увеличить client_max_body_size в конфиге Angie/Nginx.');
    console.groupEnd();
    return;
  }

  if (response.status === 504) {
    console.error('БЛОКИРУЕТ TIMEOUT ANGIE/NGINX: 504 Gateway Timeout');
    console.warn('Нужно увеличить fastcgi_read_timeout / proxy_read_timeout или ускорять обработку.');
    console.groupEnd();
    return;
  }

  if (response.status === 502) {
    console.error('Проблема PHP-FPM/Angie: 502 Bad Gateway');
    console.warn('Возможные причины: PHP-FPM упал, request_terminate_timeout, memory_limit, fatal error.');
    console.groupEnd();
    return;
  }

  if (!json) {
    const clean = String(text || '')
      .replace(/<script[\s\S]*?<\/script>/gi, ' ')
      .replace(/<style[\s\S]*?<\/style>/gi, ' ')
      .replace(/<[^>]*>/g, ' ')
      .replace(/\s+/g, ' ')
      .trim()
      .slice(0, 1000);

    console.error('Сервер вернул НЕ JSON.');
    console.log('Ответ сервера:', clean);

    if (/post_max_size|upload_max_filesize|maximum allowed size/i.test(clean)) {
      console.error('Скорее всего блокирует PHP: post_max_size / upload_max_filesize.');
    } else if (/Request Entity Too Large|413/i.test(clean)) {
      console.error('Скорее всего блокирует Angie/Nginx: client_max_body_size.');
    } else if (/Gateway Timeout|504/i.test(clean)) {
      console.error('Скорее всего timeout Angie/Nginx или PHP-FPM.');
    } else {
      console.warn('Скорее всего PHP fatal/error или страница авторизации вместо JSON. Смотри network response и PHP error_log.');
    }

    console.groupEnd();
    return;
  }

  console.log('JSON response:', json);

  if (!json.ok) {
    const msg = String(json.message || json.error || '');

    console.error('API вернул ошибку:', msg);

    if (/MAX_FILE_SIZE|FILE_TOO_LARGE|maxFileSize|too large|размер/i.test(msg)) {
      console.error('Скорее всего блокирует НАШ КОМПОНЕНТ или PHP upload limit.');
      console.warn('Проверь maxFileSize компонента, upload_max_filesize и post_max_size.');
    } else if (/UPLOAD_ERR_INI_SIZE|upload_max_filesize/i.test(msg)) {
      console.error('Блокирует PHP: upload_max_filesize.');
    } else if (/UPLOAD_ERR_FORM_SIZE/i.test(msg)) {
      console.error('Блокирует HTML/PHP MAX_FILE_SIZE.');
    } else if (/POST|post_max_size/i.test(msg)) {
      console.error('Блокирует PHP: post_max_size.');
    } else {
      console.warn('Ошибка из backend. Смотри message/details выше.');
    }

    console.groupEnd();
    return;
  }

  console.log('✅ Тестовая загрузка прошла. Значит этот размер НЕ блокируется Angie/PHP/компонентом.');

  try {
    await comp.loadFolder(comp.state.currentFolderId || comp.state.rootFolderId);

    const uploaded = (comp.state.items || []).find(function (item) {
      return String(item.name || '') === testFileName;
    });

    if (uploaded && uploaded.id) {
      const delPayload = comp.getBasePayload();

      delPayload.items = [{
        id: Number(uploaded.id),
        entityType: 'file'
      }];

      delPayload.sessid = comp.getSessid();

      const delRes = await comp.api('delete', delPayload);

      if (delRes && delRes.ok) {
        console.log('Тестовый файл удалён:', testFileName);
        await comp.loadFolder(comp.state.currentFolderId || comp.state.rootFolderId);
      } else {
        console.warn('Тестовый файл загружен, но удалить автоматически не получилось:', testFileName, delRes);
      }
    } else {
      console.warn('Тестовый файл загружен, но не найден для автоудаления:', testFileName);
    }
  } catch (cleanupError) {
    console.warn('Ошибка автоудаления тестового файла:', cleanupError);
  }

  console.groupEnd();
})();

Как читать результат:

413 → блокирует Angie/Nginx, нужен client_max_body_size
504 → таймаут Angie/Nginx/PHP-FPM
JSON с ошибкой размера → компонент Disk или PHP
Загрузка прошла → этот размер не блокируется

Для проверки начни с 60 МБ, потом 100, 200, 500.