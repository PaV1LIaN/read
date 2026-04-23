Понял. Значит программный .click() по скрытому элементу не поднимает viewer в твоей сборке.

В таком случае надо сделать ровно как в рабочем примере:
для docx/xlsx/pptx кнопка Открыть должна быть не <button>, а сам viewer-элемент с data-viewer, который пользователь нажимает напрямую.

Что поменять в script.js

Файл:

/local/sitebuilder/components/disk/script.js


---

1. В renderItemsTable() замени формирование openControl

Найди внутри метода renderItemsTable() строку:

var openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';

И замени на это:

var openControl = '';

if (item.entityType === 'folder') {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
} else if (item.previewMode === 'office') {
  var actions = escapeHtml(JSON.stringify([{ type: 'download' }]));

  openControl =
    '<span ' +
      'class="sb-disk__row-btn sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
      'data-viewer="" ' +
      'data-viewer-type="cloud-document" ' +
      'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
      'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
      'data-viewer-extension="disk.viewer.document-item" ' +
      'data-object-id="' + escapeHtml(item.id) + '" ' +
      'data-title="' + escapeHtml(item.name) + '" ' +
      'data-actions="' + actions + '"' +
    '>Открыть</span>';
} else {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
}


---

2. В renderItemsGrid() тоже замени openControl

Найди внутри renderItemsGrid() строку:

var openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';

И замени на это:

var openControl = '';

if (item.entityType === 'folder') {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
} else if (item.previewMode === 'office') {
  var actions = escapeHtml(JSON.stringify([{ type: 'download' }]));

  openControl =
    '<span ' +
      'class="sb-disk__row-btn sb-disk__viewer-btn disk-detail-sidebar-editor-item disk-detail-sidebar-editor-item-show" ' +
      'data-viewer="" ' +
      'data-viewer-type="cloud-document" ' +
      'data-src="' + escapeHtml(item.previewUrl || '') + '" ' +
      'data-viewer-type-class="BX.Disk.Viewer.DocumentItem" ' +
      'data-viewer-extension="disk.viewer.document-item" ' +
      'data-object-id="' + escapeHtml(item.id) + '" ' +
      'data-title="' + escapeHtml(item.name) + '" ' +
      'data-actions="' + actions + '"' +
    '>Открыть</span>';
} else {
  openControl = '<button type="button" class="sb-disk__row-btn" data-row-action="open">Открыть</button>';
}


---

3. Убери специальную JS-логику для office из обработчика open

Найди блок:

if (previewMode === 'office') {
  var hiddenViewer = row.querySelector('.sb-disk__office-viewer-trigger');
  if (hiddenViewer) {
    hiddenViewer.click();
    return;
  }

  if (previewUrl) {
    window.open(previewUrl, '_blank');
  }
  return;
}

И замени на:

if (previewMode === 'office') {
  return;
}

Почему так: для office-файла viewer теперь откроется сам по клику на span, а не через наш JS.


---

4. Удали из renderItemsTable() и renderItemsGrid() весь код с:

officeViewerHtml

Он больше не нужен.


---

5. Добавь стиль, чтобы span выглядел как кнопка

Файл:

/local/sitebuilder/components/disk/styles.css

Добавь в конец:

.sb-disk__viewer-btn {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  cursor: pointer;
  user-select: none;
}


---

Очень важно

На странице должен быть подключен viewer core.
В public_page.php внутри блока для диска должно остаться:

<?php if ($pageHasDiskBlock): ?>
<?php CJSCore::Init(['ui.viewer']); ?>
<script src="/bitrix/js/main/core/core.js"></script>
<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js"></script>
<?php endif; ?>


---

Почему это правильнее

Твой рабочий код с файлами лаборатории открывает документы не через JS API напрямую, а через реальный DOM-элемент с data-viewer.
Значит и тут надо делать тот же паттерн, а не эмулировать его.


---

Что сделать сейчас

1. заменить openControl в двух методах


2. удалить officeViewerHtml и логику hidden click


3. добавить CSS для sb-disk__viewer-btn


4. Ctrl + F5



Если после этого все равно не откроется, пришли HTML кнопки Открыть у одного docx из инспектора браузера.