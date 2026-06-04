Да, делаем так:

1. Файлы и папки открываются двойным кликом.
2. Кнопка “Открыть” у файлов и папок исчезает.
3. Подсказка “двойной клик для открытия” исчезает.
4. Скачать / Переим. / Удалить остаются.

1. В script.js замени функцию renderOpenControl

Файл:

/local/sitebuilder/components/disk/script.js

Найди функцию:

function renderOpenControl(item) {

и замени её целиком на:

function renderOpenControl(item) {
  if (item.entityType === 'folder') {
    return '';
  }

  if (item.previewMode === 'office') {
    return '' +
      '<span ' +
        'class="sb-disk__hidden-viewer disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
        'data-viewer="" ' +
        'data-row-action="open" ' +
        'data-viewer-type="cloud-document" ' +
        'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
        'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
        'data-viewer-extension="disk.viewer.document-item" ' +
        'data-object-id="' + escapeHtml(item.id) + '" ' +
        'data-title="' + escapeHtml(item.name) + '" ' +
        'data-actions="' + escapeHtml(JSON.stringify([{ type: 'download' }])) + '"' +
      '></span>';
  }

  return '';
}

Так для doc/docx скрытый viewer-элемент останется в HTML, чтобы двойной клик мог открыть файл через Bitrix Viewer, но кнопки видно не будет.


---

2. Проверь двойной клик

У тебя должен быть метод:

DiskComponent.prototype.openFileFromElement = function (element) {

Если внутри него есть этот кусок:

if (previewMode === 'office') {
  var viewerBtn = element.querySelector('[data-viewer]');

  if (viewerBtn) {
    viewerBtn.click();
    return;
  }
}

Оставь его. Он теперь будет нажимать скрытый viewer-элемент.


---

3. В styles.css добавь в самый конец

Файл:

/local/sitebuilder/components/disk/styles.css

/* =========================================================
   DISK: hide open buttons and double-click hints
   ========================================================= */

/* Убираем текстовые подсказки "двойной клик для открытия" */
.sb-disk__row[data-entity-type="folder"] .sb-disk__modern-name-title::after,
.sb-disk__row[data-entity-type="file"] .sb-disk__modern-name-title::after,
.sb-disk__card[data-entity-type="folder"] .sb-disk__card-name::after,
.sb-disk__card[data-entity-type="file"] .sb-disk__card-name::after {
    content: none !important;
    display: none !important;
}

/* Скрытый элемент для Bitrix Viewer у office-файлов */
.sb-disk__hidden-viewer {
    position: absolute !important;
    width: 1px !important;
    height: 1px !important;
    min-width: 1px !important;
    min-height: 1px !important;
    padding: 0 !important;
    margin: 0 !important;
    border: 0 !important;
    overflow: hidden !important;
    opacity: 0 !important;
    pointer-events: none !important;
}

/* На всякий случай скрываем старые кнопки Открыть, если где-то остались */
.sb-disk__row-btn[data-row-action="open"]:not(.sb-disk__hidden-viewer),
.sb-disk__viewer-btn[data-row-action="open"]:not(.sb-disk__hidden-viewer) {
    display: none !important;
}


---

4. Обнови версии подключения

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=13"></script>

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/disk/styles.css?v=14">

После этого сделай Ctrl + F5.

Теперь в действиях останется примерно так:

Скачать | Переим. | Удалить

А открытие будет только двойным кликом по строке или карточке.