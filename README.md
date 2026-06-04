Сделаем так: один клик по файлу ничего не открывает, а двойной клик по строке файла или карточке файла открывает файл.

Для папок оставим как сейчас: папка открывается одним кликом.


---

1. В script.js добавь метод открытия файла

Файл:

/local/sitebuilder/components/disk/script.js

Найди место перед методом:

DiskComponent.prototype.bindStaticEvents = function () {

И перед ним вставь:

DiskComponent.prototype.openFileFromElement = function (element) {
  if (!element) {
    return;
  }

  var entityType = element.getAttribute('data-entity-type') || '';

  if (entityType !== 'file') {
    return;
  }

  var previewMode = element.getAttribute('data-preview-mode') || '';
  var previewUrl = element.getAttribute('data-preview-url') || '';
  var downloadUrl = element.getAttribute('data-download-url') || '';

  if (previewMode === 'office') {
    var viewerBtn = element.querySelector('[data-viewer]');

    if (viewerBtn) {
      viewerBtn.click();
      return;
    }
  }

  if (previewUrl) {
    window.open(previewUrl, '_blank');
    return;
  }

  if (downloadUrl) {
    window.open(downloadUrl, '_blank');
  }
};


---

2. В bindStaticEvents() добавь обработчик двойного клика

Внутри метода:

DiskComponent.prototype.bindStaticEvents = function () {
  var self = this;

Сразу после строки:

var self = this;

добавь:

this.root.addEventListener('dblclick', function (e) {
  var item = e.target.closest(
    '.sb-disk__row[data-id][data-entity-type="file"], .sb-disk__card[data-id][data-entity-type="file"]'
  );

  if (!item || !self.root.contains(item)) {
    return;
  }

  if (e.target.closest('button, input, label, a, [data-viewer]')) {
    return;
  }

  self.openFileFromElement(item);
});


---

3. В styles.css добавь визуальную подсказку

Файл:

/local/sitebuilder/components/disk/styles.css

В конец добавь:

/* =========================================================
   DOUBLE CLICK OPEN FILE
   ========================================================= */

.sb-disk__row[data-entity-type="file"],
.sb-disk__card[data-entity-type="file"] {
    cursor: default;
}

.sb-disk__row[data-entity-type="file"] .sb-disk__name-cell,
.sb-disk__card[data-entity-type="file"] .sb-disk__card-name {
    cursor: pointer;
}

.sb-disk__row[data-entity-type="file"] .sb-disk__modern-name-title::after {
    content: " · двойной клик для открытия";
    color: #9ca3af;
    font-size: 11px;
    font-weight: 600;
}

.sb-disk__card[data-entity-type="file"] .sb-disk__card-name::after {
    content: "Двойной клик для открытия";
    display: block;
    margin-top: 4px;
    color: #9ca3af;
    font-size: 11px;
    font-weight: 600;
}


---

4. Обнови версию подключения

В public_page.php обнови версии:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=11"></script>

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=12">

После этого сделай Ctrl + F5.

Теперь файл будет открываться так:

двойной клик по строке файла
двойной клик по карточке файла
кнопка “Открыть” работает как раньше

Для doc/docx будет пытаться открывать через Bitrix Viewer, для png/pdf — через previewUrl, если он есть, иначе через downloadUrl.