Да, это лучше сделать прямо в script.js, чтобы независимо от сортировки API папки всегда отображались первыми.

1. Добавь метод getDisplayItems

Файл:

/local/sitebuilder/components/disk/script.js

Найди метод:

DiskComponent.prototype.renderSubtitle = function () {

Перед ним вставь:

DiskComponent.prototype.getDisplayItems = function () {
  var folders = [];
  var files = [];

  this.state.items.forEach(function (item) {
    if (String(item.entityType || '').toLowerCase() === 'folder') {
      folders.push(item);
    } else {
      files.push(item);
    }
  });

  return folders.concat(files);
};


---

2. В renderItemsTable() замени вывод

Найди:

tbody.innerHTML = this.state.items.map(function (item) {

Замени на:

tbody.innerHTML = this.getDisplayItems().map(function (item) {


---

3. В renderItemsGrid() тоже замени вывод

Найди:

container.innerHTML = this.state.items.map(function (item) {

Замени на:

container.innerHTML = this.getDisplayItems().map(function (item) {


---

Что получится

Теперь в таблице и в плитке порядок будет такой:

Папки
Папки
Папки
Файлы
Файлы
Файлы

При этом сортировка Сначала новые / По названию / По типу останется, но внутри двух групп:

сначала сортируются папки,
потом сортируются файлы.

После замены обнови подключение скрипта в public_page.php, например:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=10"></script>

И сделай Ctrl + F5.