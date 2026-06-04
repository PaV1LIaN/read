Да, значит дело не просто в двух dblclick внутри файла. Скорее всего компонент диска инициализируется два раза или где-то остался старый обработчик. Сделаем защиту на двух уровнях:

1. запретим повторную инициализацию .sb-disk;


2. поставим глобальный lock на открытие файла, чтобы даже если обработчик сработал дважды, вкладка открылась один раз.




---

1. В script.js замени функцию openFileFromElement

Найди:

DiskComponent.prototype.openFileFromElement = function (element) {

и замени всю функцию целиком:

DiskComponent.prototype.openFileFromElement = function (element) {
  if (!element) {
    return;
  }

  var entityType = element.getAttribute('data-entity-type') || '';

  if (entityType !== 'file') {
    return;
  }

  var entityId = Number(element.getAttribute('data-id') || 0);
  var previewMode = element.getAttribute('data-preview-mode') || '';
  var previewUrl = element.getAttribute('data-preview-url') || '';
  var downloadUrl = element.getAttribute('data-download-url') || '';

  var openUrl = previewUrl || downloadUrl || '';
  var openKey = 'file:' + entityId + ':' + openUrl;
  var now = Date.now();

  window.__SB_DISK_OPEN_LOCK__ = window.__SB_DISK_OPEN_LOCK__ || {
    key: '',
    time: 0
  };

  if (
    window.__SB_DISK_OPEN_LOCK__.key === openKey &&
    now - window.__SB_DISK_OPEN_LOCK__.time < 1200
  ) {
    return;
  }

  window.__SB_DISK_OPEN_LOCK__.key = openKey;
  window.__SB_DISK_OPEN_LOCK__.time = now;

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

2. В bindStaticEvents() оставь один обработчик двойного клика

Внутри:

DiskComponent.prototype.bindStaticEvents = function () {
  var self = this;

сразу после:

var self = this;

вставь вот этот обработчик:

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

  if (typeof e.stopImmediatePropagation === 'function') {
    e.stopImmediatePropagation();
  }

  var entityType = item.getAttribute('data-entity-type') || '';
  var entityId = Number(item.getAttribute('data-id') || 0);

  var clickKey = entityType + ':' + entityId;
  var now = Date.now();

  window.__SB_DISK_DBLCLICK_LOCK__ = window.__SB_DISK_DBLCLICK_LOCK__ || {
    key: '',
    time: 0
  };

  if (
    window.__SB_DISK_DBLCLICK_LOCK__.key === clickKey &&
    now - window.__SB_DISK_DBLCLICK_LOCK__.time < 1200
  ) {
    return;
  }

  window.__SB_DISK_DBLCLICK_LOCK__.key = clickKey;
  window.__SB_DISK_DBLCLICK_LOCK__.time = now;

  if (entityType === 'folder') {
    if (entityId > 0) {
      await self.loadFolder(entityId);
    }

    return;
  }

  if (entityType === 'file') {
    self.openFileFromElement(item);
  }
}, true);

Важно: последний параметр true нужен, чтобы обработчик сработал раньше остальных и остановил лишние открытия.


---

3. Удали остальные dblclick

В этом же файле найди все остальные места:

addEventListener('dblclick'

Должен остаться только один обработчик — тот, который выше.


---

4. Внизу script.js защити компонент от повторной инициализации

В самом низу файла найди:

document.addEventListener('DOMContentLoaded', function () {
  document.querySelectorAll('.sb-disk').forEach(function (root) {
    var component = new DiskComponent(root);
    component.init();
  });
});

Замени на:

document.addEventListener('DOMContentLoaded', function () {
  document.querySelectorAll('.sb-disk').forEach(function (root) {
    if (root.getAttribute('data-disk-component-ready') === '1') {
      return;
    }

    root.setAttribute('data-disk-component-ready', '1');

    var component = new DiskComponent(root);
    component.init();
  });
});


---

5. Обнови версию скрипта

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=15"></script>

После этого сделай Ctrl + F5.

Если после этого всё равно откроются 2 вкладки, значит script.js подключён на странице два раза. Тогда в исходном коде страницы нужно найти два подключения:

components/disk/script.js

и оставить только одно.