Да, сделаем нормальное модальное окно прогресса загрузки.

Будет так:

1. Выбираешь файл / перетаскиваешь файл.
2. Открывается модальное окно “Загрузка файлов”.
3. Показывается список файлов.
4. Показывается общий прогресс в процентах.
5. После загрузки пишет “Загрузка завершена”.
6. Потом обновляет список файлов.


---

1. Правим script.js

Файл:

/local/sitebuilder/components/disk/script.js


---

1.1. Добавь метод загрузки с прогрессом

Найди метод:

DiskComponent.prototype.api = async function (action, payload, isFormData) {

После всего этого метода, сразу после его закрытия, вставь:

DiskComponent.prototype.apiUploadWithProgress = function (action, formData, onProgress) {
  return new Promise(function (resolve, reject) {
    var xhr = new XMLHttpRequest();

    xhr.open(
      'POST',
      '/local/sitebuilder/components/disk/api.php?action=' + encodeURIComponent(action),
      true
    );

    xhr.upload.addEventListener('progress', function (event) {
      if (!event.lengthComputable) {
        return;
      }

      if (typeof onProgress === 'function') {
        onProgress({
          loaded: event.loaded,
          total: event.total,
          percent: event.total > 0 ? Math.round((event.loaded / event.total) * 100) : 0
        });
      }
    });

    xhr.addEventListener('load', function () {
      var text = xhr.responseText || '';
      var json = null;

      try {
        json = JSON.parse(text);
      } catch (e) {
        reject(new Error('UPLOAD_BAD_RESPONSE'));
        return;
      }

      resolve(json);
    });

    xhr.addEventListener('error', function () {
      reject(new Error('UPLOAD_NETWORK_ERROR'));
    });

    xhr.addEventListener('abort', function () {
      reject(new Error('UPLOAD_ABORTED'));
    });

    xhr.send(formData);
  });
};


---

2. Добавь методы модального окна

Найди секцию:

/* =========================================================
   EVENTS
   ========================================================= */

Сразу перед ней вставь:

/* =========================================================
   UPLOAD STATUS MODAL
   ========================================================= */

DiskComponent.prototype.ensureUploadStatusModal = function () {
  if (this.uploadStatusModal && document.body.contains(this.uploadStatusModal)) {
    return this.uploadStatusModal;
  }

  var modal = document.createElement('div');

  modal.className = 'sb-disk-upload-status-modal';
  modal.hidden = true;

  modal.innerHTML = ''
    + '<div class="sb-disk-upload-status-modal__backdrop"></div>'
    + '<div class="sb-disk-upload-status-modal__dialog">'
    + '  <div class="sb-disk-upload-status-modal__head">'
    + '    <div>'
    + '      <div class="sb-disk-upload-status-modal__title">Загрузка файлов</div>'
    + '      <div class="sb-disk-upload-status-modal__subtitle" data-upload-status-subtitle>Подготовка...</div>'
    + '    </div>'
    + '    <button type="button" class="sb-disk-upload-status-modal__close" data-upload-status-close hidden>×</button>'
    + '  </div>'
    + ''
    + '  <div class="sb-disk-upload-status-modal__body">'
    + '    <div class="sb-disk-upload-progress">'
    + '      <div class="sb-disk-upload-progress__top">'
    + '        <span data-upload-status-message>Подготовка файлов...</span>'
    + '        <strong data-upload-status-percent>0%</strong>'
    + '      </div>'
    + '      <div class="sb-disk-upload-progress__track">'
    + '        <div class="sb-disk-upload-progress__bar" data-upload-status-bar></div>'
    + '      </div>'
    + '      <div class="sb-disk-upload-progress__size" data-upload-status-size>0 Б / 0 Б</div>'
    + '    </div>'
    + ''
    + '    <div class="sb-disk-upload-file-list" data-upload-status-files></div>'
    + '  </div>'
    + '</div>';

  var closeBtn = modal.querySelector('[data-upload-status-close]');

  if (closeBtn) {
    closeBtn.addEventListener('click', function () {
      modal.hidden = true;
    });
  }

  document.body.appendChild(modal);

  this.uploadStatusModal = modal;

  return modal;
};

DiskComponent.prototype.showUploadStatusModal = function (files) {
  files = Array.prototype.slice.call(files || []);

  var modal = this.ensureUploadStatusModal();
  var subtitle = modal.querySelector('[data-upload-status-subtitle]');
  var message = modal.querySelector('[data-upload-status-message]');
  var percent = modal.querySelector('[data-upload-status-percent]');
  var bar = modal.querySelector('[data-upload-status-bar]');
  var size = modal.querySelector('[data-upload-status-size]');
  var list = modal.querySelector('[data-upload-status-files]');
  var closeBtn = modal.querySelector('[data-upload-status-close]');

  var totalSize = files.reduce(function (sum, file) {
    return sum + Number(file && file.size ? file.size : 0);
  }, 0);

  if (subtitle) {
    subtitle.textContent = 'Файлов: ' + files.length;
  }

  if (message) {
    message.textContent = 'Начинаю загрузку...';
  }

  if (percent) {
    percent.textContent = '0%';
  }

  if (bar) {
    bar.style.width = '0%';
  }

  if (size) {
    size.textContent = '0 Б / ' + formatBytes(totalSize);
  }

  if (list) {
    list.innerHTML = files.map(function (file) {
      return ''
        + '<div class="sb-disk-upload-file">'
        + '  <div class="sb-disk-upload-file__name">' + escapeHtml(file.name || 'file') + '</div>'
        + '  <div class="sb-disk-upload-file__size">' + escapeHtml(formatBytes(file.size || 0)) + '</div>'
        + '</div>';
    }).join('');
  }

  if (closeBtn) {
    closeBtn.hidden = true;
  }

  modal.classList.remove('is-success', 'is-error');
  modal.hidden = false;
};

DiskComponent.prototype.updateUploadStatusModal = function (data) {
  data = data || {};

  var modal = this.ensureUploadStatusModal();
  var message = modal.querySelector('[data-upload-status-message]');
  var percent = modal.querySelector('[data-upload-status-percent]');
  var bar = modal.querySelector('[data-upload-status-bar]');
  var size = modal.querySelector('[data-upload-status-size]');

  var loaded = Number(data.loaded || 0);
  var total = Number(data.total || 0);
  var progress = Number(data.percent || 0);

  if (progress < 0) {
    progress = 0;
  }

  if (progress > 100) {
    progress = 100;
  }

  if (message) {
    message.textContent = data.message || 'Загружаю файлы...';
  }

  if (percent) {
    percent.textContent = progress + '%';
  }

  if (bar) {
    bar.style.width = progress + '%';
  }

  if (size) {
    size.textContent = formatBytes(loaded) + ' / ' + formatBytes(total);
  }
};

DiskComponent.prototype.finishUploadStatusModal = function (success, messageText) {
  var modal = this.ensureUploadStatusModal();
  var message = modal.querySelector('[data-upload-status-message]');
  var percent = modal.querySelector('[data-upload-status-percent]');
  var bar = modal.querySelector('[data-upload-status-bar]');
  var closeBtn = modal.querySelector('[data-upload-status-close]');

  modal.classList.toggle('is-success', !!success);
  modal.classList.toggle('is-error', !success);

  if (message) {
    message.textContent = messageText || (success ? 'Загрузка завершена' : 'Ошибка загрузки');
  }

  if (success) {
    if (percent) {
      percent.textContent = '100%';
    }

    if (bar) {
      bar.style.width = '100%';
    }

    setTimeout(function () {
      modal.hidden = true;
    }, 900);
  } else if (closeBtn) {
    closeBtn.hidden = false;
  }
};


---

3. Замени функцию uploadFiles

В твоём script.js найди функцию:

DiskComponent.prototype.uploadFiles = async function (files) {

И замени её целиком на эту:

DiskComponent.prototype.uploadFiles = async function (files) {
  files = Array.prototype.slice.call(files || []);

  if (!files.length) {
    return;
  }

  if (!this.state.permissions.canUpload) {
    alert('У вас нет прав на загрузку файлов');
    return;
  }

  var preparedFiles = [];

  try {
    for (var i = 0; i < files.length; i++) {
      var file = files[i];

      if (!file || !file.name) {
        continue;
      }

      var existingItem = this.findExistingFileByName(file.name);

      if (!existingItem) {
        preparedFiles.push(file);
        continue;
      }

      var decision = await this.askDuplicateUploadAction(file, existingItem);

      if (!decision || decision.action === 'cancel') {
        continue;
      }

      if (decision.action === 'replace') {
        await this.archiveExistingFileToHistory(existingItem);

        preparedFiles.push(file);
        continue;
      }

      if (decision.action === 'rename') {
        preparedFiles.push(this.makeRenamedFile(file, decision.name));
      }
    }

    if (!preparedFiles.length) {
      return;
    }

    this.showUploadStatusModal(preparedFiles);

    var formData = new FormData();

    formData.append('siteId', this.state.siteId);
    formData.append('pageId', this.state.pageId);
    formData.append('blockId', this.state.blockId);
    formData.append('currentFolderId', this.state.currentFolderId);
    formData.append('sessid', this.getSessid());

    preparedFiles.forEach(function (file) {
      formData.append('files[]', file);
    });

    var self = this;

    var res = await this.apiUploadWithProgress('upload', formData, function (progress) {
      self.updateUploadStatusModal({
        loaded: progress.loaded,
        total: progress.total,
        percent: progress.percent,
        message: 'Загружаю файлы...'
      });
    });

    if (!res || !res.ok) {
      this.finishUploadStatusModal(false, (res && (res.message || res.error)) || 'Ошибка загрузки');
      return;
    }

    this.updateUploadStatusModal({
      loaded: 1,
      total: 1,
      percent: 100,
      message: 'Загрузка завершена. Обновляю список...'
    });

    await this.loadFolder(this.state.currentFolderId);

    this.finishUploadStatusModal(true, 'Загрузка завершена');
  } catch (err) {
    console.error(err);

    this.finishUploadStatusModal(false, err && err.message ? err.message : 'Ошибка загрузки');
  }
};


---

4. Добавь CSS

Файл:

/local/sitebuilder/components/disk/styles.css

В конец добавь:

/* =========================================================
   Disk upload status modal
   ========================================================= */

.sb-disk-upload-status-modal[hidden] {
    display: none !important;
}

.sb-disk-upload-status-modal {
    position: fixed;
    inset: 0;
    z-index: 99999;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
}

.sb-disk-upload-status-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(15, 23, 42, .55);
    backdrop-filter: blur(4px);
}

.sb-disk-upload-status-modal__dialog {
    position: relative;
    width: min(520px, 100%);
    max-height: min(680px, calc(100vh - 48px));
    overflow: auto;
    background: #fff;
    border-radius: 22px;
    border: 1px solid rgba(226, 232, 240, .9);
    box-shadow: 0 24px 80px rgba(15, 23, 42, .28);
}

.sb-disk-upload-status-modal__head {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    padding: 20px 22px 14px;
    border-bottom: 1px solid rgba(226, 232, 240, .9);
}

.sb-disk-upload-status-modal__title {
    color: #0f172a;
    font-size: 18px;
    font-weight: 900;
    line-height: 1.2;
}

.sb-disk-upload-status-modal__subtitle {
    margin-top: 4px;
    color: #64748b;
    font-size: 13px;
    font-weight: 700;
}

.sb-disk-upload-status-modal__close {
    width: 34px;
    height: 34px;
    border: 0;
    border-radius: 12px;
    background: #f1f5f9;
    color: #334155;
    font-size: 22px;
    line-height: 1;
    cursor: pointer;
}

.sb-disk-upload-status-modal__body {
    padding: 18px 22px 22px;
}

.sb-disk-upload-progress {
    padding: 14px;
    border-radius: 16px;
    background: #f8fafc;
    border: 1px solid rgba(226, 232, 240, .95);
}

.sb-disk-upload-progress__top {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 10px;
    color: #334155;
    font-size: 13px;
    font-weight: 800;
}

.sb-disk-upload-progress__top strong {
    color: #1d4ed8;
    font-size: 13px;
    font-weight: 900;
}

.sb-disk-upload-progress__track {
    height: 10px;
    overflow: hidden;
    border-radius: 999px;
    background: #e2e8f0;
}

.sb-disk-upload-progress__bar {
    height: 100%;
    width: 0;
    border-radius: inherit;
    background: linear-gradient(90deg, #2563eb, #38bdf8);
    transition: width .18s ease;
}

.sb-disk-upload-progress__size {
    margin-top: 8px;
    color: #64748b;
    font-size: 12px;
    font-weight: 800;
}

.sb-disk-upload-file-list {
    display: grid;
    gap: 8px;
    margin-top: 14px;
}

.sb-disk-upload-file {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    padding: 10px 12px;
    border-radius: 12px;
    background: #fff;
    border: 1px solid rgba(226, 232, 240, .95);
}

.sb-disk-upload-file__name {
    min-width: 0;
    overflow: hidden;
    color: #0f172a;
    font-size: 13px;
    font-weight: 800;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-disk-upload-file__size {
    flex: 0 0 auto;
    color: #64748b;
    font-size: 12px;
    font-weight: 800;
}

.sb-disk-upload-status-modal.is-success .sb-disk-upload-progress__bar {
    background: linear-gradient(90deg, #16a34a, #22c55e);
}

.sb-disk-upload-status-modal.is-error .sb-disk-upload-progress__bar {
    background: linear-gradient(90deg, #dc2626, #fb7185);
}

.sb-disk-upload-status-modal.is-error .sb-disk-upload-progress__top strong {
    color: #dc2626;
}


---

5. Проверка

После правок:

1. Ctrl + F5.
2. Открой страницу с компонентом Диск.
3. Нажми “Загрузить”.
4. Выбери большой файл, чтобы прогресс было видно.

Должно появиться модальное окно с прогрессом загрузки.