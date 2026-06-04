Да, проблема понятна: файл истории создался, но наш код ищет историю только по префиксу:

_history__

А у тебя в диске он создался как:

history__178...

или похожий вариант без первого _.

Из-за этого файл истории не скрывается и не привязывается к кнопке “История”.

Нужно сделать поддержку обоих вариантов и дальше создавать историю уже без первого подчёркивания.


---

1. В script.js замени методы истории

Файл:

/local/sitebuilder/components/disk/script.js

Найди этот блок:

DiskComponent.prototype.isHistoryItem = function (item) {

И замени все методы от isHistoryItem до buildHistoryFileName включительно на этот код:

DiskComponent.prototype.getHistoryNameInfo = function (item) {
  var name = String(item && item.name ? item.name : '');

  var prefixes = [
    '_history__',
    'history__',
    '_history_',
    'history_'
  ];

  for (var i = 0; i < prefixes.length; i++) {
    var prefix = prefixes[i];

    if (name.indexOf(prefix) !== 0) {
      continue;
    }

    var rest = name.slice(prefix.length);
    var sepIndex = rest.indexOf('__');

    if (sepIndex < 0) {
      return {
        isHistory: true,
        originalName: name,
        timestamp: 0
      };
    }

    var timePart = rest.slice(0, sepIndex);
    var originalName = rest.slice(sepIndex + 2);
    var timestamp = Number(String(timePart).split('_')[0] || 0);

    return {
      isHistory: true,
      originalName: originalName || name,
      timestamp: timestamp > 0 ? timestamp : 0
    };
  }

  return {
    isHistory: false,
    originalName: name,
    timestamp: 0
  };
};

DiskComponent.prototype.isHistoryItem = function (item) {
  return this.getHistoryNameInfo(item).isHistory;
};

DiskComponent.prototype.getHistoryOriginalName = function (item) {
  return this.getHistoryNameInfo(item).originalName;
};

DiskComponent.prototype.getHistoryTime = function (item) {
  return this.getHistoryNameInfo(item).timestamp;
};

DiskComponent.prototype.formatHistoryDate = function (timestamp) {
  timestamp = Number(timestamp || 0);

  if (!timestamp) {
    return 'Старая версия';
  }

  var date = new Date(timestamp);

  return date.toLocaleString('ru-RU');
};

DiskComponent.prototype.buildHistoryFileName = function (originalName) {
  originalName = String(originalName || '').trim();

  if (!originalName) {
    originalName = 'file';
  }

  var stamp = Date.now() + '_' + Math.floor(Math.random() * 100000);

  return 'history__' + stamp + '__' + originalName;
};


---

2. Проверь, что getDisplayItems() такой

Найди:

DiskComponent.prototype.getDisplayItems = function () {

Должно быть так:

DiskComponent.prototype.getDisplayItems = function () {
  var folders = [];
  var files = [];

  this.state.items.forEach(function (item) {
    if (this.isHistoryItem(item)) {
      return;
    }

    if (String(item.entityType || '').toLowerCase() === 'folder') {
      folders.push(item);
    } else {
      files.push(item);
    }
  }, this);

  return folders.concat(files);
};


---

3. Проверь, что findExistingFileByName() игнорирует историю

Должно быть так:

DiskComponent.prototype.findExistingFileByName = function (fileName) {
  fileName = String(fileName || '').trim().toLowerCase();

  if (!fileName) {
    return null;
  }

  for (var i = 0; i < this.state.items.length; i++) {
    var item = this.state.items[i];

    if (this.isHistoryItem(item)) {
      continue;
    }

    if (String(item.entityType || '').toLowerCase() !== 'file') {
      continue;
    }

    if (String(item.name || '').trim().toLowerCase() === fileName) {
      return item;
    }
  }

  return null;
};


---

4. Обнови версию подключения

В public_page.php:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=18"></script>

Потом сделай Ctrl + F5.

После этого файл вида:

history__1780570953891_53726__1.png

перестанет отображаться в общем списке, а у нового 1.png должна появиться кнопка “История”.