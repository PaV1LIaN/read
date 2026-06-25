Да, можно. Сейчас 90% висит долго по двум причинам:

1. Старый “фейковый” таймер распаковки сам догоняет прогресс до 90%.
2. Один шаг распаковки может обрабатывать несколько файлов, и пока запрос идёт — окно не знает, какой файл сейчас внутри сервера.

Сделаем нормально:

1. Уберём фейковый таймер.
2. Backend будет отдавать имя текущего/следующего файла.
3. JS будет писать: “Распаковываю: имя_файла”.
4. Шаг распаковки сделаем по 1 файлу за запрос — прогресс будет чаще обновляться.


---

1. Правим unpack_archive_start.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_start.php

В самом конце найди:

DiskResponse::success([
    'jobId' => $jobId,
    'archiveName' => (string)$file->getName(),

И перед DiskResponse::success добавь:

$firstEntryName = '';

if (!empty($entries[0]['rawName'])) {
    $firstEntryName = basename((string)$entries[0]['rawName']);
}

Теперь сам ответ замени на такой:

DiskResponse::success([
    'jobId' => $jobId,
    'archiveName' => (string)$file->getName(),
    'nextEntryName' => $firstEntryName,
    'targetFolder' => [
        'id' => (int)$targetFolder->getId(),
        'name' => (string)$targetFolder->getName(),
    ],
    'totalEntries' => count($entries),
    'totalFiles' => $totalFiles,
    'totalFolders' => $totalFolders,
    'totalSize' => $totalSize,
]);


---

2. Правим unpack_archive_step.php

Файл:

/local/sitebuilder/components/disk/actions/unpack_archive_step.php

2.1. Сделай по 1 файлу за шаг

Найди:

$batchLimit = 5;

Замени на:

$batchLimit = 1;


---

2.2. Добавь переменную имени текущего файла

Найди:

$processedThisStep = 0;

Сразу после добавь:

$lastEntryName = '';


---

2.3. Внутри цикла сохрани имя файла

Найди внутри while:

$rawName = (string)($entry['rawName'] ?? '');
$isDir = !empty($entry['isDir']);
$size = (int)($entry['size'] ?? 0);

Сразу после добавь:

$lastEntryName = basename($rawName);


---

2.4. Перед ответом добавь следующий файл

Найди ближе к концу:

if ($done) {
    $percent = 100;
    sb_disk_unpack_delete_job($jobId);
} else {
    sb_disk_unpack_save_job($job);
}

Сразу после этого добавь:

$nextEntryName = '';

if (!$done && isset($entries[$index]) && is_array($entries[$index])) {
    $nextEntryName = basename((string)($entries[$index]['rawName'] ?? ''));
}


---

2.5. В ответ добавь имена файлов

Найди ответ:

DiskResponse::success([
    'jobId' => $jobId,
    'done' => $done,

Внутрь массива добавь:

'lastEntryName' => $lastEntryName,
    'nextEntryName' => $nextEntryName,

Итоговый начало ответа должно быть так:

DiskResponse::success([
    'jobId' => $jobId,
    'done' => $done,
    'index' => $index,
    'totalEntries' => $totalEntries,
    'percent' => $percent,
    'lastEntryName' => $lastEntryName,
    'nextEntryName' => $nextEntryName,


---

3. Правим script.js

Файл:

/local/sitebuilder/components/disk/script.js


---

3.1. Убери фейковый таймер из showUnpackStatusModal

Найди в функции:

DiskComponent.prototype.showUnpackStatusModal = function (fileName) {

В конце функции найди строку:

this.startUnpackProgressTicker();

Замени на:

// Реальный прогресс теперь приходит с сервера через unpackArchiveStep.
// Фейковый таймер больше не запускаем.
this.stopUnpackProgressTicker();


---

3.2. Замени метод unpackArchiveFromRow

Найди:

DiskComponent.prototype.unpackArchiveFromRow = async function (unpackRow) {

Замени весь метод целиком на этот:

DiskComponent.prototype.unpackArchiveFromRow = async function (unpackRow) {
  var fileName = unpackRow.getAttribute('data-name') || 'архив';

  this.showUnpackStatusModal(fileName);

  this.updateUnpackStatusModal({
    percent: 1,
    message: 'Создаю задачу распаковки...',
    info: 'Проверяю архив.'
  });

  var startPayload = this.getBasePayload();

  startPayload.fileId = Number(unpackRow.getAttribute('data-id') || 0);
  startPayload.sessid = this.getSessid();

  var startRes = await this.api('unpackArchiveStart', startPayload);

  if (!startRes || !startRes.ok) {
    this.finishUnpackStatusModal(
      false,
      (startRes && (startRes.message || startRes.error)) || 'Не удалось начать распаковку',
      {}
    );
    return;
  }

  var startData = startRes.data || {};
  var jobId = startData.jobId || '';

  if (!jobId) {
    this.finishUnpackStatusModal(false, 'UNPACK_JOB_ID_EMPTY', {});
    return;
  }

  var totalEntries = Number(startData.totalEntries || 0);
  var totalFiles = Number(startData.totalFiles || 0);
  var totalFolders = Number(startData.totalFolders || 0);
  var nextEntryName = startData.nextEntryName || '';

  this.updateUnpackStatusModal({
    percent: 1,
    message: 'Архив проверен. Начинаю распаковку...',
    info:
      'Файлов: ' + totalFiles +
      ' · Папок: ' + totalFolders +
      ' · Всего элементов: ' + totalEntries
  });

  var done = false;
  var lastData = null;

  while (!done) {
    if (nextEntryName) {
      this.updateUnpackStatusModal({
        percent: lastData && lastData.percent ? lastData.percent : 1,
        message: 'Распаковываю: ' + nextEntryName,
        info:
          'Обработано: ' +
          (lastData && lastData.index ? lastData.index : 0) +
          ' из ' +
          totalEntries
      });
    }

    var stepPayload = this.getBasePayload();

    stepPayload.jobId = jobId;
    stepPayload.sessid = this.getSessid();

    var stepRes = await this.api('unpackArchiveStep', stepPayload);

    if (!stepRes || !stepRes.ok) {
      this.finishUnpackStatusModal(
        false,
        (stepRes && (stepRes.message || stepRes.error)) || 'Ошибка шага распаковки',
        {}
      );
      return;
    }

    var stepData = stepRes.data || {};

    lastData = stepData;
    done = !!stepData.done;
    nextEntryName = stepData.nextEntryName || '';

    var percent = Number(stepData.percent || 1);

    if (percent < 1) {
      percent = 1;
    }

    if (percent > 100) {
      percent = 100;
    }

    this.updateUnpackStatusModal({
      percent: percent,
      message: done
        ? 'Завершаю распаковку...'
        : 'Распаковано: ' + (stepData.lastEntryName || 'элемент'),
      info:
        'Обработано: ' +
        (stepData.index || 0) +
        ' из ' +
        (stepData.totalEntries || totalEntries) +
        ' · Файлов: ' +
        (stepData.extractedFiles || 0) +
        ' · Папок: ' +
        (stepData.createdFolders || 0)
    });

    if (!done) {
      await this.sleep(120);
    }
  }

  this.finishUnpackStatusModal(true, 'Распаковка завершена', {
    extractedFiles: lastData && lastData.extractedFiles ? lastData.extractedFiles : 0,
    createdFolders: lastData && lastData.createdFolders ? lastData.createdFolders : 0,
    totalSize: lastData && lastData.totalSize ? lastData.totalSize : 0
  });

  var targetFolder = lastData && lastData.targetFolder ? lastData.targetFolder : null;
  var self = this;

  setTimeout(async function () {
    if (targetFolder && targetFolder.id) {
      await self.loadFolder(Number(targetFolder.id));
    } else {
      await self.loadFolder(self.state.currentFolderId || self.state.rootFolderId);
    }
  }, 700);
};


---

4. Почему раньше висело на 90%

Потому что эта функция:

startUnpackProgressTicker()

рисовала не настоящий прогресс, а имитацию. Она специально доходила до 90% и ждала окончания серверного запроса.

Теперь прогресс будет настоящий:

1 / 12
2 / 12
3 / 12
...
12 / 12 = 100%

И будет видно имя файла:

Распаковываю: image.png
Распаковано: document.docx


---

После правок сделай Ctrl + F5 и попробуй тот же архив.