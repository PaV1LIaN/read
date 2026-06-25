Это всё ещё не ошибка архива. Это значит, что unpackArchive возвращает HTML вместо JSON. Скорее всего ты поменял api.php, но в script.js у тебя ещё старый метод api, который делает response.json() и поэтому показывает Unexpected token.

Сделай сейчас две точные правки.


---

1. Замени метод api в script.js

Файл:

/local/sitebuilder/components/disk/script.js

Найди:

DiskComponent.prototype.api = async function (action, payload, isFormData) {

И замени весь метод целиком на этот:

DiskComponent.prototype.api = async function (action, payload, isFormData) {
  var url = '/local/sitebuilder/components/disk/api.php?action=' + encodeURIComponent(action);

  var response;

  if (isFormData) {
    response = await fetch(url, {
      method: 'POST',
      body: payload
    });
  } else {
    response = await fetch(url, {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json'
      },
      body: JSON.stringify(payload || {})
    });
  }

  var text = await response.text();
  var json = null;

  try {
    json = JSON.parse(text);
  } catch (e) {
    console.error('Disk API returned non JSON for action=' + action, text);

    var cleanText = String(text || '')
      .replace(/<script[\s\S]*?<\/script>/gi, ' ')
      .replace(/<style[\s\S]*?<\/style>/gi, ' ')
      .replace(/<[^>]*>/g, ' ')
      .replace(/\s+/g, ' ')
      .trim()
      .slice(0, 500);

    throw new Error(
      'API вернул HTML вместо JSON. action=' + action + '. Ответ: ' + cleanText
    );
  }

  return json;
};

После этого ошибка в окне станет не Unexpected token, а нормальный текст: что именно сервер вернул.


---

2. Замени блок unpackBtn

В этом же файле найди блок:

var unpackBtn = e.target.closest('[data-row-action="unpack"]');

И замени весь блок распаковки на этот:

var unpackBtn = e.target.closest('[data-row-action="unpack"]');

if (unpackBtn) {
  var unpackRow = e.target.closest('[data-id][data-entity-type="file"]');

  if (!unpackRow) {
    return;
  }

  var fileName = unpackRow.getAttribute('data-name') || 'архив';

  var confirmUnpack = window.confirm(
    'Распаковать архив "' + fileName + '"?\n\n' +
    'Файлы будут распакованы в текущую папку.'
  );

  if (!confirmUnpack) {
    return;
  }

  try {
    self.setLoading(true);
    self.showUnpackStatusModal(fileName);

    var unpackPayload = self.getBasePayload();

    unpackPayload.fileId = Number(unpackRow.getAttribute('data-id') || 0);
    unpackPayload.currentFolderId = self.state.currentFolderId || self.state.rootFolderId;
    unpackPayload.sessid = self.getSessid();

    var unpackRes = await self.api('unpackArchive', unpackPayload);

    if (!unpackRes || !unpackRes.ok) {
      self.finishUnpackStatusModal(
        false,
        (unpackRes && (unpackRes.message || unpackRes.error)) || 'Ошибка распаковки',
        {}
      );
      return;
    }

    var unpackData = unpackRes.data || {};
    var targetFolder = unpackData.targetFolder || null;

    self.finishUnpackStatusModal(true, 'Распаковка завершена', {
      extractedFiles: unpackData.extractedFiles || 0,
      createdFolders: unpackData.createdFolders || 0,
      totalSize: unpackData.totalSize || 0
    });

    setTimeout(async function () {
      if (targetFolder && targetFolder.id) {
        await self.loadFolder(Number(targetFolder.id));
      } else {
        await self.loadFolder(self.state.currentFolderId || self.state.rootFolderId);
      }
    }, 700);
  } catch (err) {
    console.error(err);

    self.finishUnpackStatusModal(
      false,
      err && err.message ? err.message : 'Ошибка распаковки',
      {}
    );
  } finally {
    self.setLoading(false);
  }

  return;
}


---

3. Проверь, что api.php точно заменён

Файл:

/local/sitebuilder/components/disk/api.php

В самом верху должно быть примерно так:

<?php

$action = (string)($_GET['action'] ?? '');

$sbDiskShouldBuffer = $action !== 'download';

if ($sbDiskShouldBuffer) {
    ob_start();
}

Если у тебя сверху всё ещё так:

<?php

require_once __DIR__ . '/bootstrap.php';

$action = (string)($_GET['action'] ?? '');

значит api.php не заменён, и поэтому сервер всё ещё отдаёт HTML при fatal-ошибке.


---

После этого сделай Ctrl + F5 и попробуй распаковать снова.

Если снова будет ошибка — она уже должна быть нормальной, например:

API вернул HTML вместо JSON. action=unpackArchive. Ответ: ...

Скинь этот текст, и я скажу точную причину. Сейчас мы пока видим только симптом: HTML вместо JSON.