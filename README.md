Да, сейчас уже видно две проблемы:

1. визуал у блока сырой


2. навигация по папкам либо неочевидная, либо не до конца работает



По скриншоту видно еще и третий баг:

3. у тебя одновременно показываются таблица и плитка, хотя должен быть только один режим.



Ниже даю, что лучше исправить сразу.


---

Что сейчас ломает UX

1. Одновременно видны таблица и плитка

Это значит, что переключение вида работает не до конца на уровне CSS/JS.

2. Сообщение “Не удалось загрузить содержимое” висит поверх уже загруженных файлов

Это значит, что состояние error не сбрасывается после успешной загрузки списка.

3. Вход в папку неудобный

Сейчас, скорее всего, папка открывается только по маленькой кнопке Открыть, а не по имени/карточке. Это неудобно и легко кажется “не работает”.

4. Нет явного индикатора, в какой папке ты находишься

Если breadcrumbs скрыты, то пользователь вообще не понимает, что он вошел внутрь.


---

Что я предлагаю сделать

Сейчас лучше сделать 3 точечных правки:

1. починить переключение table/grid


2. починить показ состояний


3. сделать открытие папки по клику на строку/карточку и всегда показывать breadcrumbs




---

1. Правка styles.css

Файл

/local/sitebuilder/components/disk/styles.css

Добавь в конец файла:

.sb-disk [hidden] {
  display: none !important;
}

.sb-disk {
  border: 1px solid #e5e7eb;
  border-radius: 18px;
  background: #ffffff;
  padding: 18px;
  box-shadow: 0 8px 24px rgba(15, 23, 42, 0.06);
}

.sb-disk__header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  gap: 16px;
  margin-bottom: 16px;
}

.sb-disk__title {
  margin: 0;
  font-size: 22px;
  line-height: 1.2;
  font-weight: 700;
  color: #111827;
}

.sb-disk__subtitle {
  margin: 4px 0 0;
  color: #6b7280;
  font-size: 13px;
}

.sb-disk__toolbar {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-bottom: 14px;
}

.sb-disk__toolbar-right {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  margin-left: auto;
}

.sb-disk__breadcrumbs {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px;
  margin-bottom: 14px;
}

.sb-disk__crumb {
  border: 0;
  background: #eef2ff;
  color: #3730a3;
  border-radius: 999px;
  padding: 6px 10px;
  cursor: pointer;
  font-size: 12px;
  font-weight: 600;
}

.sb-disk__states {
  margin-bottom: 12px;
}

.sb-disk__state {
  padding: 14px 16px;
  border-radius: 12px;
  border: 1px dashed #d1d5db;
  background: #f9fafb;
  color: #6b7280;
}

.sb-disk__table-wrap {
  overflow-x: auto;
}

.sb-disk__table {
  width: 100%;
  border-collapse: collapse;
}

.sb-disk__table th,
.sb-disk__table td {
  padding: 12px 10px;
  border-bottom: 1px solid #edf2f7;
  vertical-align: middle;
  text-align: left;
}

.sb-disk__row {
  transition: background .15s ease;
}

.sb-disk__row:hover {
  background: #f8fbff;
}

.sb-disk__row.is-clickable {
  cursor: pointer;
}

.sb-disk__item-name {
  display: flex;
  align-items: center;
  gap: 10px;
}

.sb-disk__item-name-label {
  font-weight: 600;
  color: #111827;
}

.sb-disk__badge {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  min-height: 24px;
  padding: 0 8px;
  border-radius: 999px;
  background: #f3f4f6;
  color: #4b5563;
  font-size: 12px;
  font-weight: 600;
}

.sb-disk__actions {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}

.sb-disk__grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(220px, 1fr));
  gap: 14px;
}

.sb-disk__card {
  border: 1px solid #e5e7eb;
  border-radius: 14px;
  background: #fff;
  padding: 14px;
  transition: box-shadow .15s ease, border-color .15s ease, transform .15s ease;
}

.sb-disk__card:hover {
  border-color: #c7d2fe;
  box-shadow: 0 10px 20px rgba(37, 99, 235, 0.08);
  transform: translateY(-1px);
}

.sb-disk__card.is-clickable {
  cursor: pointer;
}

.sb-disk__card-name {
  font-weight: 700;
  color: #111827;
  margin: 10px 0 8px;
  word-break: break-word;
}

.sb-disk__card-sub {
  color: #6b7280;
  font-size: 12px;
}

.sb-disk__card-actions {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
  margin-top: 12px;
}

.sb-disk__row-btn,
.sb-disk__view-btn,
.sb-disk__toolbar .sb-btn,
.sb-disk__toolbar-right .sb-btn {
  border-radius: 10px;
}

.sb-disk__bulkbar {
  display: flex;
  align-items: center;
  gap: 10px;
  margin-bottom: 12px;
  padding: 10px 12px;
  border-radius: 12px;
  background: #f8fafc;
  border: 1px solid #e5e7eb;
}


---

2. Правка script.js — нормальный сброс состояний и открытие папки по клику

Файл

/local/sitebuilder/components/disk/script.js

Замени renderState() на это:

DiskComponent.prototype.renderState = function (stateName) {
  var nodes = this.root.querySelectorAll('[data-state]');
  nodes.forEach(function (node) {
    node.hidden = true;
  });

  if (!stateName) {
    return;
  }

  var node = this.root.querySelector('[data-state="' + stateName + '"]');
  if (node) {
    node.hidden = false;
  }
};


---

В loadFolder() после успешной загрузки оставь только такое завершение:

Найди кусок:

if (!this.state.items.length) {
  this.renderState('empty');
} else {
  this.renderState(null);
}

и убедись, что он именно такой.
Если там есть еще error/loading логика рядом — оставь только это.


---

Сделай строку папки кликабельной

В renderItemsTable() замени <tr ...> на:

'<tr class="sb-disk__row ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +


---

Сделай карточку папки кликабельной

В renderItemsGrid() замени <div class="sb-disk__card" на:

'<div class="sb-disk__card ' + (item.entityType === 'folder' ? 'is-clickable' : '') + '" ' +


---

Добавь открытие папки по клику на строку/карточку

В bindStaticEvents() внутри обработчика this.root.addEventListener('click', async function (e) { ... })

в самое начало обработчика вставь:

var clickableRow = e.target.closest('.sb-disk__row[data-id][data-entity-type="folder"], .sb-disk__card[data-id][data-entity-type="folder"]');
if (clickableRow && !e.target.closest('button, input, label, a')) {
  var folderId = Number(clickableRow.getAttribute('data-id') || 0);
  if (folderId > 0) {
    await self.loadFolder(folderId);
  }
  return;
}


---

3. Всегда показывать breadcrumbs

Сейчас у тебя в bootstrap response:

"showBreadcrumbs": false

Для нормального UX папок я бы принудительно включил.

Быстрый путь

Обнови props блока в БД:

UPDATE sitebuilder.block
SET props_json = jsonb_set(props_json, '{showBreadcrumbs}', 'true'::jsonb, true)
WHERE id = 6;


---

4. Почему у тебя “не работает вход в папку для загрузки в нее”

С высокой вероятностью он либо:

открывается только по кнопке Открыть

либо открывается, но ты этого не видишь, потому что нет breadcrumbs и нет явного заголовка текущей папки


После правок выше:

можно будет открыть папку кликом по строке/карточке

будут breadcrumbs

загрузка будет идти в currentFolderId, который обновляется в loadFolder(folderId)


То есть загрузка в текущую папку начнет быть понятной пользователю.


---

Что сделать сейчас по шагам

1. вставить CSS-блок в styles.css


2. поправить renderState() в script.js


3. добавить клик по строке/карточке папки


4. включить showBreadcrumbs = true


5. Ctrl + F5




---

Что должно стать после этого

исчезнет ощущение “сломано”

не будут одновременно торчать таблица и плитка

уйдет зависшее “Не удалось загрузить содержимое”

вход в папку станет понятным

загрузка будет идти в открытую папку


Если хочешь, следующим сообщением я пришлю тебе полностью готовый обновленный script.js целиком, уже с этими правками внутри, чтобы не искать куски вручную.