Да, делаем на стороне компонента диска:

если файл с таким именем уже есть:
1. Заменить — старый файл удаляется, новый загружается с тем же именем
2. Переименовать — новый файл загружается с новым именем
3. Отмена — этот файл не загружается


---

1. script.js — добавь методы

Файл:

/local/sitebuilder/components/disk/script.js

Найди место перед:

DiskComponent.prototype.bindStaticEvents = function () {

И вставь туда:

DiskComponent.prototype.findExistingFileByName = function (fileName) {
  fileName = String(fileName || '').trim().toLowerCase();

  if (!fileName) {
    return null;
  }

  for (var i = 0; i < this.state.items.length; i++) {
    var item = this.state.items[i];

    if (String(item.entityType || '').toLowerCase() !== 'file') {
      continue;
    }

    if (String(item.name || '').trim().toLowerCase() === fileName) {
      return item;
    }
  }

  return null;
};

DiskComponent.prototype.splitFileName = function (fileName) {
  fileName = String(fileName || '').trim();

  var dotIndex = fileName.lastIndexOf('.');

  if (dotIndex <= 0) {
    return {
      base: fileName,
      ext: ''
    };
  }

  return {
    base: fileName.slice(0, dotIndex),
    ext: fileName.slice(dotIndex)
  };
};

DiskComponent.prototype.suggestDuplicateFileName = function (fileName) {
  var parts = this.splitFileName(fileName);
  var base = parts.base || 'file';
  var ext = parts.ext || '';
  var index = 1;
  var candidate = base + ' (копия)' + ext;

  while (this.findExistingFileByName(candidate)) {
    index++;
    candidate = base + ' (копия ' + index + ')' + ext;
  }

  return candidate;
};

DiskComponent.prototype.makeRenamedFile = function (file, newName) {
  newName = String(newName || '').trim();

  if (!newName) {
    throw new Error('EMPTY_FILE_NAME');
  }

  if (typeof File === 'function') {
    return new File([file], newName, {
      type: file.type,
      lastModified: file.lastModified
    });
  }

  throw new Error('Ваш браузер не поддерживает переименование файла перед загрузкой');
};

DiskComponent.prototype.deleteDiskItems = async function (items) {
  var payload = this.getBasePayload();

  payload.items = items;
  payload.sessid = this.getSessid();

  var res = await this.api('delete', payload);

  if (!res || !res.ok) {
    throw new Error((res && (res.message || res.error)) || 'DELETE_ERROR');
  }

  return res;
};

DiskComponent.prototype.askDuplicateUploadAction = function (file, existingItem) {
  var self = this;

  return new Promise(function (resolve) {
    var fileName = String(file && file.name ? file.name : '');
    var suggestedName = self.suggestDuplicateFileName(fileName);

    var modal = document.createElement('div');
    modal.className = 'sb-disk-duplicate-modal';

    modal.innerHTML = ''
      + '<div class="sb-disk-duplicate-modal__backdrop" data-duplicate-action="cancel"></div>'
      + '<div class="sb-disk-duplicate-modal__dialog">'
      + '  <div class="sb-disk-duplicate-modal__head">'
      + '    <div>'
      + '      <div class="sb-disk-duplicate-modal__title">Файл уже существует</div>'
      + '      <div class="sb-disk-duplicate-modal__subtitle">В этой папке уже есть файл с таким именем.</div>'
      + '    </div>'
      + '    <button type="button" class="sb-disk-duplicate-modal__close" data-duplicate-action="cancel">×</button>'
      + '  </div>'
      + ''
      + '  <div class="sb-disk-duplicate-modal__body">'
      + '    <div class="sb-disk-duplicate-file">'
      + '      <div class="sb-disk-duplicate-file__label">Файл:</div>'
      + '      <div class="sb-disk-duplicate-file__name">' + escapeHtml(fileName) + '</div>'
      + '    </div>'
      + ''
      + '    <div class="sb-disk-duplicate-field">'
      + '      <label>Новое имя, если выбрать “Переименовать”</label>'
      + '      <input type="text" class="sb-disk-duplicate-input" value="' + escapeHtml(suggestedName) + '">'
      + '    </div>'
      + ''
      + '    <div class="sb-disk-duplicate-note">'
      + '      “Заменить” удалит старый файл и загрузит новый с тем же именем.'
      + '    </div>'
      + '  </div>'
      + ''
      + '  <div class="sb-disk-duplicate-modal__footer">'
      + '    <button type="button" class="sb-disk-duplicate-btn" data-duplicate-action="cancel">Отмена</button>'
      + '    <button type="button" class="sb-disk-duplicate-btn" data-duplicate-action="rename">Переименовать</button>'
      + '    <button type="button" class="sb-disk-duplicate-btn is-primary" data-duplicate-action="replace">Заменить</button>'
      + '  </div>'
      + '</div>';

    function close(result) {
      if (modal && modal.parentNode) {
        modal.parentNode.removeChild(modal);
      }

      document.removeEventListener('keydown', onKeyDown);

      resolve(result);
    }

    function onKeyDown(e) {
      if (e.key === 'Escape') {
        close({
          action: 'cancel'
        });
      }
    }

    modal.addEventListener('click', function (e) {
      var btn = e.target.closest('[data-duplicate-action]');

      if (!btn) {
        return;
      }

      var action = btn.getAttribute('data-duplicate-action');

      if (action === 'cancel') {
        close({
          action: 'cancel'
        });
        return;
      }

      if (action === 'replace') {
        close({
          action: 'replace',
          existingItem: existingItem
        });
        return;
      }

      if (action === 'rename') {
        var input = modal.querySelector('.sb-disk-duplicate-input');
        var newName = input ? String(input.value || '').trim() : '';

        if (!newName) {
          alert('Введите новое имя файла');
          if (input) input.focus();
          return;
        }

        if (self.findExistingFileByName(newName)) {
          alert('Файл с таким именем уже есть. Укажите другое имя.');
          if (input) input.focus();
          return;
        }

        close({
          action: 'rename',
          name: newName
        });
      }
    });

    document.addEventListener('keydown', onKeyDown);
    document.body.appendChild(modal);

    setTimeout(function () {
      var input = modal.querySelector('.sb-disk-duplicate-input');
      if (input) {
        input.focus();
        input.select();
      }
    }, 50);
  });
};


---

2. script.js — замени обработчик загрузки

Найди внутри bindStaticEvents() вот этот блок:

uploadInput.addEventListener('change', async function (e) {
  var files = Array.prototype.slice.call(e.target.files || []);
  if (!files.length) {
    return;
  }

  var formData = new FormData();
  formData.append('siteId', self.state.siteId);
  formData.append('pageId', self.state.pageId);
  formData.append('blockId', self.state.blockId);
  formData.append('currentFolderId', self.state.currentFolderId);
  formData.append('sessid', self.getSessid());

  files.forEach(function (file) {
    formData.append('files[]', file);
  });

  var res = await self.api('upload', formData, true);
  if (!res || !res.ok) {
    window.alert((res && (res.message || res.error)) || 'Ошибка загрузки');
    return;
  }

  uploadInput.value = '';
  await self.loadFolder(self.state.currentFolderId);
});

Замени его целиком на:

uploadInput.addEventListener('change', async function (e) {
  var files = Array.prototype.slice.call(e.target.files || []);

  if (!files.length) {
    return;
  }

  if (!self.state.permissions.canUpload) {
    uploadInput.value = '';
    return;
  }

  var preparedFiles = [];

  try {
    for (var i = 0; i < files.length; i++) {
      var file = files[i];
      var existingItem = self.findExistingFileByName(file.name);

      if (!existingItem) {
        preparedFiles.push(file);
        continue;
      }

      var decision = await self.askDuplicateUploadAction(file, existingItem);

      if (!decision || decision.action === 'cancel') {
        continue;
      }

      if (decision.action === 'replace') {
        await self.deleteDiskItems([{
          id: Number(existingItem.id || 0),
          entityType: 'file'
        }]);

        self.state.items = self.state.items.filter(function (item) {
          return Number(item.id || 0) !== Number(existingItem.id || 0);
        });

        preparedFiles.push(file);
        continue;
      }

      if (decision.action === 'rename') {
        preparedFiles.push(self.makeRenamedFile(file, decision.name));
      }
    }

    if (!preparedFiles.length) {
      uploadInput.value = '';
      return;
    }

    var formData = new FormData();

    formData.append('siteId', self.state.siteId);
    formData.append('pageId', self.state.pageId);
    formData.append('blockId', self.state.blockId);
    formData.append('currentFolderId', self.state.currentFolderId);
    formData.append('sessid', self.getSessid());

    preparedFiles.forEach(function (file) {
      formData.append('files[]', file);
    });

    var res = await self.api('upload', formData, true);

    if (!res || !res.ok) {
      window.alert((res && (res.message || res.error)) || 'Ошибка загрузки');
      return;
    }

    uploadInput.value = '';
    await self.loadFolder(self.state.currentFolderId);
  } catch (err) {
    console.error(err);
    uploadInput.value = '';
    window.alert(err && err.message ? err.message : 'Ошибка загрузки');
  }
});


---

3. styles.css — добавь стили окна выбора

В конец файла:

/local/sitebuilder/components/disk/styles.css

добавь:

/* =========================================================
   DUPLICATE FILE UPLOAD MODAL
   ========================================================= */

.sb-disk-duplicate-modal {
    position: fixed;
    inset: 0;
    z-index: 20000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
}

.sb-disk-duplicate-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(15, 23, 42, .48);
    backdrop-filter: blur(5px);
}

.sb-disk-duplicate-modal__dialog {
    position: relative;
    width: min(520px, 100%);
    overflow: hidden;
    border: 1px solid #e5e7eb;
    border-radius: 22px;
    background: #fff;
    box-shadow: 0 28px 80px rgba(15, 23, 42, .30);
}

.sb-disk-duplicate-modal__head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 16px;
    padding: 20px 22px;
    border-bottom: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-disk-duplicate-modal__title {
    color: #111827;
    font-size: 20px;
    font-weight: 900;
    line-height: 1.25;
}

.sb-disk-duplicate-modal__subtitle {
    margin-top: 5px;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.45;
}

.sb-disk-duplicate-modal__close {
    width: 34px;
    height: 34px;
    border: 1px solid #e5e7eb;
    border-radius: 11px;
    background: #fff;
    color: #6b7280;
    cursor: pointer;
    font-size: 22px;
    line-height: 1;
}

.sb-disk-duplicate-modal__close:hover {
    background: #f3f4f6;
    color: #111827;
}

.sb-disk-duplicate-modal__body {
    padding: 22px;
}

.sb-disk-duplicate-file {
    padding: 12px;
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    background: #f8fafc;
}

.sb-disk-duplicate-file__label {
    color: #6b7280;
    font-size: 12px;
    font-weight: 700;
}

.sb-disk-duplicate-file__name {
    margin-top: 4px;
    color: #111827;
    font-size: 14px;
    font-weight: 900;
    word-break: break-word;
}

.sb-disk-duplicate-field {
    margin-top: 14px;
}

.sb-disk-duplicate-field label {
    display: block;
    margin-bottom: 6px;
    color: #374151;
    font-size: 13px;
    font-weight: 800;
}

.sb-disk-duplicate-input {
    width: 100%;
    height: 40px;
    padding: 0 13px;
    border: 1px solid #dbe3ef;
    border-radius: 13px;
    background: #fff;
    color: #111827;
    font-size: 13px;
    outline: none;
}

.sb-disk-duplicate-input:focus {
    border-color: var(--disk-accent, #2563eb);
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12);
}

.sb-disk-duplicate-note {
    margin-top: 12px;
    padding: 10px 12px;
    border: 1px solid #dbeafe;
    border-radius: 12px;
    background: #eff6ff;
    color: #1e40af;
    font-size: 12px;
    line-height: 1.45;
}

.sb-disk-duplicate-modal__footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 22px;
    border-top: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-disk-duplicate-btn {
    min-height: 38px;
    padding: 0 14px;
    border: 1px solid #dbe3ef;
    border-radius: 12px;
    background: #fff;
    color: #374151;
    font-size: 13px;
    font-weight: 800;
    cursor: pointer;
}

.sb-disk-duplicate-btn:hover {
    border-color: #c7d2fe;
    background: #eef2ff;
    color: var(--disk-accent, #2563eb);
}

.sb-disk-duplicate-btn.is-primary {
    border-color: var(--disk-accent, #2563eb);
    background: var(--disk-accent, #2563eb);
    color: #fff;
}

.sb-disk-duplicate-btn.is-primary:hover {
    background: var(--disk-accent-hover, #3f5de9);
    border-color: var(--disk-accent-hover, #3f5de9);
    color: #fff;
}

@media (max-width: 620px) {
    .sb-disk-duplicate-modal {
        align-items: flex-start;
        padding: 12px;
    }

    .sb-disk-duplicate-modal__footer {
        flex-direction: column-reverse;
    }

    .sb-disk-duplicate-btn {
        width: 100%;
    }
}


---

4. Обнови версии подключения

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=16"></script>

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=15">

После этого сделай Ctrl + F5.

Теперь при загрузке файла с уже существующим именем появится окно с выбором: Заменить / Переименовать / Отмена.