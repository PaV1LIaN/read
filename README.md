Проблема почти точно в том, что сейчас в script.js осталось два обработчика dblclick:

1. первый мы добавляли для файлов;


2. второй потом добавили для файлов и папок.



Из-за этого при двойном клике по картинке срабатывают оба обработчика, и открываются 2 вкладки.

Нужно оставить только один обработчик двойного клика.


---

1. В script.js удали все старые dblclick

Файл:

/local/sitebuilder/components/disk/script.js

Найди внутри bindStaticEvents() все блоки вида:

this.root.addEventListener('dblclick', function (e) {

и удали их полностью.

То есть в файле не должно остаться старых dblclick-обработчиков.


---

2. Вставь один правильный обработчик

Внутри:

DiskComponent.prototype.bindStaticEvents = function () {
  var self = this;

сразу после:

var self = this;

вставь:

this.root.addEventListener('dblclick', async function (e) {
  var item = e.target.closest(
    '.sb-disk__row[data-id][data-entity-type], .sb-disk__card[data-id][data-entity-type]'
  );

  if (!item || !self.root.contains(item)) {
    return;
  }

  if (e.target.closest('button, input, label, a')) {
    return;
  }

  e.preventDefault();
  e.stopPropagation();

  var entityType = item.getAttribute('data-entity-type') || '';
  var entityId = Number(item.getAttribute('data-id') || 0);

  var openKey = entityType + ':' + entityId;
  var now = Date.now();

  if (self._lastDoubleOpenKey === openKey && now - self._lastDoubleOpenAt < 700) {
    return;
  }

  self._lastDoubleOpenKey = openKey;
  self._lastDoubleOpenAt = now;

  if (entityType === 'folder') {
    if (entityId > 0) {
      await self.loadFolder(entityId);
    }

    return;
  }

  if (entityType === 'file') {
    self.openFileFromElement(item);
  }
});


---

3. Проверь openFileFromElement

Функция должна быть одна. Если её нет — добавь перед bindStaticEvents():

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

4. Обнови версию скрипта

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=14"></script>

После этого Ctrl + F5.

Теперь при двойном клике должна открываться только одна вкладка.