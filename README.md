Это уже не ошибка подключения. Сейчас проблема в другом:

**миграция ничего не нашла в JSON**, поэтому все массивы пустые и получилось:

* `sites migrated: 0`
* `pages migrated: 0`
* `blocks migrated: 0`
* и так далее

То есть скрипт **отработал**, но читал **не из того пути**.

`current_schema = public` тут не главный симптом. Для твоего скрипта важнее, что он взял пустые данные.

---

# Что это значит

В моем скрипте было:

```php
$storageDir = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/storage';
```

А у тебя, скорее всего, JSON лежат **не в `/storage/`**, а в другом каталоге проекта.

---

# Что сделать прямо сейчас

## 1. Вставь в `migrate_json_to_pg.php` такой debug-блок

Сразу после строки:

```php
$storageDir = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/storage';
```

вставь:

```php
echo '<pre>';
echo "DOCUMENT_ROOT: " . $_SERVER['DOCUMENT_ROOT'] . PHP_EOL;
echo "storageDir: " . $storageDir . PHP_EOL;

$filesToCheck = [
    'sites.json',
    'pages.json',
    'blocks.json',
    'access.json',
    'menus.json',
    'layouts.json',
];

foreach ($filesToCheck as $fileName) {
    $fullPath = $storageDir . '/' . $fileName;
    echo $fullPath . ' => ' . (is_file($fullPath) ? 'FOUND' : 'NOT FOUND') . PHP_EOL;
}
echo '</pre>';
```

И снова открой `migrate_json_to_pg.php`.

---

## 2. Скорее всего увидишь `NOT FOUND`

Тогда надо просто поправить путь.

---

# Какой путь проверить первым

Судя по структуре `sitebuilder`, очень вероятны такие варианты:

```php
$storageDir = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/data';
```

или

```php
$storageDir = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/storage/data';
```

или

```php
$storageDir = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder';
```

---

# Самый быстрый способ найти JSON

Сделай временный файл, например:

## `/local/sitebuilder/tools/find_json.php`

```php
<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

$root = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder';

echo '<pre>';

$iterator = new RecursiveIteratorIterator(
    new RecursiveDirectoryIterator($root, FilesystemIterator::SKIP_DOTS)
);

foreach ($iterator as $file) {
    if (!$file->isFile()) {
        continue;
    }

    $name = $file->getFilename();
    if (in_array($name, ['sites.json', 'pages.json', 'blocks.json', 'access.json', 'menus.json', 'layouts.json'], true)) {
        echo $file->getPathname() . PHP_EOL;
    }
}

echo '</pre>';
```

Открой его в браузере. Он покажет, **где реально лежат JSON**.

---

# После этого

Когда найдешь путь, просто замени в `migrate_json_to_pg.php`:

```php
$storageDir = $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/storage';
```

на правильный каталог.

---

# Про `current_schema = public`

Это не обязательно проблема.

Потому что в `db.php` у тебя сейчас есть:

```php
$pdo->exec("SET search_path TO sitebuilder, public");
```

И этого достаточно, чтобы обращаться к таблицам без префикса схемы.

Если хочешь проверить точно, добавь в `test_pg.php`:

```php
$stmt = $pdo->query("show search_path");
print_r($stmt->fetch(PDO::FETCH_ASSOC));
```

Но это не причина `0 migrated`.

---

# Итого

Сейчас у тебя не проблема с PostgreSQL.
Сейчас проблема в том, что:

**скрипт миграции читает JSON не из той папки**.

Пришли вывод `find_json.php`, и я сразу скажу, какой exact путь поставить в `migrate_json_to_pg.php`.
