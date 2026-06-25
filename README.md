Да, сделаем такое же модальное окно, как при загрузке, только для распаковки ZIP.

Так как распаковка идёт одним серверным запросом, настоящий процент по каждому файлу без фоновой очереди получить сложно. Поэтому сделаем нормально для пользователя:

1. Открывается окно “Распаковка архива”.
2. Прогресс плавно идёт до 90%.
3. Когда сервер закончил — ставим 100%.
4. Показываем: сколько файлов распаковано и сколько папок создано.
5. Потом открываем созданную папку.


---

1. script.js — добавь методы модального окна

Файл:

/local/sitebuilder/components/disk/script.js

Найди секцию:

/* =========================================================
   EVENTS
   ========================================================= */

Перед ней вставь:

/* =========================================================
   UNPACK STATUS MODAL
   ========================================================= */

DiskComponent.prototype.ensureUnpackStatusModal = function () {
  if (this.unpackStatusModal && document.body.contains(this.unpackStatusModal)) {
    return this.unpackStatusModal;
  }

  var modal = document.createElement('div');

  modal.className = 'sb-disk-upload-status-modal sb-disk-unpack-status-modal';
  modal.hidden = true;

  modal.innerHTML = ''
    + '<div class="sb-disk-upload-status-modal__backdrop"></div>'
    + '<div class="sb-disk-upload-status-modal__dialog">'
    + '  <div class="sb-disk-upload-status-modal__head">'
    + '    <div>'
    + '      <div class="sb-disk-upload-status-modal__title">Распаковка архива</div>'
    + '      <div class="sb-disk-upload-status-modal__subtitle" data-unpack-status-subtitle>Подготовка...</div>'
    + '    </div>'
    + '    <button type="button" class="sb-disk-upload-status-modal__close" data-unpack-status-close hidden>×</button>'
    + '  </div>'
    + ''
    + '  <div class="sb-disk-upload-status-modal__body">'
    + '    <div class="sb-disk-upload-progress">'
    + '      <div class="sb-disk-upload-progress__top">'
    + '        <span data-unpack-status-message>Подготовка архива...</span>'
    + '        <strong data-unpack-status-percent>0%</strong>'
    + '      </div>'
    + '      <div class="sb-disk-upload-progress__track">'
    + '        <div class="sb-disk-upload-progress__bar" data-unpack-status-bar></div>'
    + '      </div>'
    + '      <div class="sb-disk-upload-progress__size" data-unpack-status-info>Ожидание...</div>'
    + '    </div>'
    + ''
    + '    <div class="sb-disk-upload-file-list">'
    + '      <div class="sb-disk-upload-file">'
    + '        <div class="sb-disk-upload-file__name" data-unpack-status-file>Архив</div>'
    + '        <div class="sb-disk-upload-file__size">ZIP</div>'
    + '      </div>'
    + '    </div>'
    + '  </div>'
    + '</div>';

  var closeBtn = modal.querySelector('[data-unpack-status-close]');

  if (closeBtn) {
    closeBtn.addEventListener('click', function () {
      modal.hidden = true;
    });
  }

  document.body.appendChild(modal);

  this.unpackStatusModal = modal;

  return modal;
};

DiskComponent.prototype.showUnpackStatusModal = function (fileName) {
  var modal = this.ensureUnpackStatusModal();

  var subtitle = modal.querySelector('[data-unpack-status-subtitle]');
  var message = modal.querySelector('[data-unpack-status-message]');
  var percent = modal.querySelector('[data-unpack-status-percent]');
  var bar = modal.querySelector('[data-unpack-status-bar]');
  var info = modal.querySelector('[data-unpack-status-info]');
  var file = modal.querySelector('[data-unpack-status-file]');
  var closeBtn = modal.querySelector('[data-unpack-status-close]');

  if (subtitle) {
    subtitle.textContent = 'Файл: ' + (fileName || 'архив.zip');
  }

  if (message) {
    message.textContent = 'Начинаю распаковку...';
  }

  if (percent) {
    percent.textContent = '0%';
  }

  if (bar) {
    bar.style.width = '0%';
  }

  if (info) {
    info.textContent = 'Создаю папку и проверяю архив...';
  }

  if (file) {
    file.textContent = fileName || 'архив.zip';
  }

  if (closeBtn) {
    closeBtn.hidden = true;
  }

  modal.classList.remove('is-success', 'is-error');
  modal.hidden = false;

  this.startUnpackProgressTicker();
};

DiskComponent.prototype.updateUnpackStatusModal = function (data) {
  data = data || {};

  var modal = this.ensureUnpackStatusModal();
  var message = modal.querySelector('[data-unpack-status-message]');
  var percent = modal.querySelector('[data-unpack-status-percent]');
  var bar = modal.querySelector('[data-unpack-status-bar]');
  var info = modal.querySelector('[data-unpack-status-info]');

  var progress = Number(data.percent || 0);

  if (progress < 0) {
    progress = 0;
  }

  if (progress > 100) {
    progress = 100;
  }

  if (message) {
    message.textContent = data.message || 'Распаковываю архив...';
  }

  if (percent) {
    percent.textContent = progress + '%';
  }

  if (bar) {
    bar.style.width = progress + '%';
  }

  if (info) {
    info.textContent = data.info || 'Пожалуйста, подождите...';
  }
};

DiskComponent.prototype.startUnpackProgressTicker = function () {
  var self = this;

  this.stopUnpackProgressTicker();

  this.unpackProgressValue = 0;

  this.unpackProgressTimer = setInterval(function () {
    var value = Number(self.unpackProgressValue || 0);

    if (value < 30) {
      value += 7;
    } else if (value < 60) {
      value += 4;
    } else if (value < 85) {
      value += 2;
    } else if (value < 90) {
      value += 1;
    } else {
      value = 90;
    }

    self.unpackProgressValue = value;

    self.updateUnpackStatusModal({
      percent: value,
      message: value < 40 ? 'Проверяю архив...' : 'Распаковываю файлы...',
      info: 'Это может занять несколько секунд.'
    });
  }, 350);
};

DiskComponent.prototype.stopUnpackProgressTicker = function () {
  if (this.unpackProgressTimer) {
    clearInterval(this.unpackProgressTimer);
    this.unpackProgressTimer = null;
  }
};

DiskComponent.prototype.finishUnpackStatusModal = function (success, messageText, data) {
  data = data || {};

  this.stopUnpackProgressTicker();

  var modal = this.ensureUnpackStatusModal();
  var message = modal.querySelector('[data-unpack-status-message]');
  var percent = modal.querySelector('[data-unpack-status-percent]');
  var bar = modal.querySelector('[data-unpack-status-bar]');
  var info = modal.querySelector('[data-unpack-status-info]');
  var closeBtn = modal.querySelector('[data-unpack-status-close]');

  modal.classList.toggle('is-success', !!success);
  modal.classList.toggle('is-error', !success);

  if (message) {
    message.textContent = messageText || (success ? 'Распаковка завершена' : 'Ошибка распаковки');
  }

  if (success) {
    if (percent) {
      percent.textContent = '100%';
    }

    if (bar) {
      bar.style.width = '100%';
    }

    if (info) {
      var extractedFiles = Number(data.extractedFiles || 0);
      var createdFolders = Number(data.createdFolders || 0);

      info.textContent = 'Файлов: ' + extractedFiles + ' · Папок: ' + createdFolders;
    }

    setTimeout(function () {
      modal.hidden = true;
    }, 1200);
  } else {
    if (info) {
      info.textContent = 'Проверьте архив или настройки сервера.';
    }

    if (closeBtn) {
      closeBtn.hidden = false;
    }
  }
};


---

2. Замени обработчик unpack

В этом же файле найди блок:

var unpackBtn = e.target.closest('[data-row-action="unpack"]');

И замени весь блок unpack на этот:

var unpackBtn = e.target.closest('[data-row-action="unpack"]');

if (unpackBtn) {
  var unpackRow = e.target.closest('[data-id][data-entity-type="file"]');

  if (!unpackRow) {
    return;
  }

  var fileName = unpackRow.getAttribute('data-name') || 'архив';

  var confirmUnpack = window.confirm(
    'Распаковать архив "' + fileName + '"?\n\n' +
    'Будет создана новая папка с содержимым архива.'
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
        (unpackRes && (unpackRes.message || unpackRes.error)) || 'Ошибка распаковки'
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
      err && err.message ? err.message : 'Ошибка распаковки'
    );
  } finally {
    self.setLoading(false);
  }

  return;
}


---

3. CSS можно почти не добавлять

Мы используем те же классы, что и окно загрузки:

sb-disk-upload-status-modal
sb-disk-upload-progress
sb-disk-upload-file

Поэтому если окно загрузки уже стилизовано, окно распаковки тоже будет красивым.

Но добавь в конец:

/local/sitebuilder/components/disk/styles.css

вот это:

/* =========================================================
   Disk unpack status modal
   ========================================================= */

.sb-disk-unpack-status-modal .sb-disk-upload-progress__bar {
    background: linear-gradient(90deg, #7c3aed, #38bdf8);
}

.sb-disk-unpack-status-modal.is-success .sb-disk-upload-progress__bar {
    background: linear-gradient(90deg, #16a34a, #22c55e);
}

.sb-disk-unpack-status-modal.is-error .sb-disk-upload-progress__bar {
    background: linear-gradient(90deg, #dc2626, #fb7185);
}


---

После этого:

1. Ctrl + F5.
2. Загрузи ZIP.
3. Нажми “Распаковать”.
4. Должно появиться окно распаковки с прогрессом.

Если хочешь, следующим шагом сделаем реальную очередь распаковки с настоящим прогрессом по файлам, но там уже нужен отдельный unpack_start.php, unpack_status.php и временная таблица/файл статуса.