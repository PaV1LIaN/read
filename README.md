Да, сделаем загрузку через drag & drop прямо на область диска.

Логика будет такая:

перетащил файл в блок диска
→ блок подсветился
→ отпустил файл
→ если такого файла нет — загрузился
→ если файл уже есть — появится окно: Заменить / Переименовать / Отмена


---

1. script.js — добавь метод загрузки файлов

Файл:

/local/sitebuilder/components/disk/script.js

Найди место перед:

DiskComponent.prototype.bindStaticEvents = function () {

и вставь туда:

DiskComponent.prototype.setDragOver = function (active) {
  this.root.classList.toggle('is-dragover', !!active);
};

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

    var formData = new FormData();

    formData.append('siteId', this.state.siteId);
    formData.append('pageId', this.state.pageId);
    formData.append('blockId', this.state.blockId);
    formData.append('currentFolderId', this.state.currentFolderId);
    formData.append('sessid', this.getSessid());

    preparedFiles.forEach(function (file) {
      formData.append('files[]', file);
    });

    var res = await this.api('upload', formData, true);

    if (!res || !res.ok) {
      window.alert((res && (res.message || res.error)) || 'Ошибка загрузки');
      return;
    }

    await this.loadFolder(this.state.currentFolderId);
  } catch (err) {
    console.error(err);
    window.alert(err && err.message ? err.message : 'Ошибка загрузки');
  }
};


---

2. script.js — замени обработчик обычной загрузки

Найди внутри bindStaticEvents() вот этот кусок:

uploadInput.addEventListener('change', async function (e) {

и замени весь обработчик change на этот:

uploadInput.addEventListener('change', async function (e) {
  var files = Array.prototype.slice.call(e.target.files || []);

  try {
    await self.uploadFiles(files);
  } finally {
    uploadInput.value = '';
  }
});

То есть старую большую логику загрузки из change убираем, потому что теперь она вынесена в общий метод uploadFiles().


---

3. script.js — добавь drag & drop обработчики

Внутри bindStaticEvents() найди блок:

var uploadBtn = this.root.querySelector('[data-action="upload"]');
var uploadInput = this.root.querySelector('[data-role="upload-input"]');

После всего блока:

if (uploadBtn && uploadInput) {
  ...
}

сразу вставь:

var dragDepth = 0;

function hasDraggedFiles(e) {
  var types = e.dataTransfer && e.dataTransfer.types;

  if (!types) {
    return false;
  }

  return Array.prototype.indexOf.call(types, 'Files') !== -1;
}

this.root.addEventListener('dragenter', function (e) {
  if (!hasDraggedFiles(e)) {
    return;
  }

  e.preventDefault();
  e.stopPropagation();

  dragDepth++;
  self.setDragOver(true);
});

this.root.addEventListener('dragover', function (e) {
  if (!hasDraggedFiles(e)) {
    return;
  }

  e.preventDefault();
  e.stopPropagation();

  if (e.dataTransfer) {
    e.dataTransfer.dropEffect = self.state.permissions.canUpload ? 'copy' : 'none';
  }

  self.setDragOver(true);
});

this.root.addEventListener('dragleave', function (e) {
  if (!hasDraggedFiles(e)) {
    return;
  }

  e.preventDefault();
  e.stopPropagation();

  dragDepth--;

  if (dragDepth <= 0) {
    dragDepth = 0;
    self.setDragOver(false);
  }
});

this.root.addEventListener('drop', async function (e) {
  if (!hasDraggedFiles(e)) {
    return;
  }

  e.preventDefault();
  e.stopPropagation();

  dragDepth = 0;
  self.setDragOver(false);

  var files = Array.prototype.slice.call(
    e.dataTransfer && e.dataTransfer.files ? e.dataTransfer.files : []
  );

  await self.uploadFiles(files);
});


---

4. styles.css — добавь оформление drag & drop

Файл:

/local/sitebuilder/components/disk/styles.css

В конец добавь:

/* =========================================================
   DRAG AND DROP UPLOAD
   ========================================================= */

.sb-disk {
    position: relative;
}

.sb-disk.is-dragover::before {
    content: "";
    position: absolute;
    inset: -8px;
    z-index: 50;
    border: 2px dashed var(--disk-accent, #2563eb);
    border-radius: 22px;
    background: rgba(37, 99, 235, .08);
    pointer-events: none;
}

.sb-disk.is-dragover::after {
    content: "Отпустите файлы для загрузки";
    position: absolute;
    left: 50%;
    top: 50%;
    z-index: 51;
    transform: translate(-50%, -50%);
    min-width: 260px;
    padding: 16px 22px;
    border-radius: 18px;
    background: #ffffff;
    color: #111827;
    font-size: 15px;
    font-weight: 900;
    text-align: center;
    box-shadow: 0 20px 60px rgba(15, 23, 42, .25);
    pointer-events: none;
}


---

5. Обнови версии подключения

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=22"></script>

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=18">

После этого сделай Ctrl + F5.

Теперь файлы можно будет загружать двумя способами:

через кнопку “Загрузить”
перетаскиванием файла с компьютера в блок диска