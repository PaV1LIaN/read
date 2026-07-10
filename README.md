Миграция прошла нормально, но есть важный момент:

[page_access] => page_access
[access] => access

— эти таблицы есть.

А вот:

[pages] =>

значит таблицы:

sitebuilder.pages

нет. Поэтому дальше PageAccessRepository::getPageAndParentIds() может упасть, потому что я заложил название sitebuilder.pages, а у тебя таблица страниц, видимо, называется по-другому.

Нужно найти реальное название таблицы страниц.


---

1. Создай временный диагностический файл

Файл:

/local/sitebuilder/migrations/check_sitebuilder_tables.php

Код:

<?php

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/db.php';

global $USER;

if (!is_object($USER) || !$USER->IsAdmin()) {
    die('ACCESS_DENIED');
}

$pdo = sb_db();

echo '<pre>';

echo "TABLES IN SCHEMA sitebuilder:\n\n";

$stmt = $pdo->query("
    SELECT table_schema, table_name
    FROM information_schema.tables
    WHERE table_schema = 'sitebuilder'
    ORDER BY table_name
");

$tables = $stmt->fetchAll(PDO::FETCH_ASSOC);

print_r($tables);

echo "\n\nCOLUMNS:\n\n";

$stmt = $pdo->query("
    SELECT table_schema, table_name, column_name, data_type
    FROM information_schema.columns
    WHERE table_schema = 'sitebuilder'
    ORDER BY table_name, ordinal_position
");

$columns = $stmt->fetchAll(PDO::FETCH_ASSOC);

$currentTable = '';

foreach ($columns as $column) {
    $table = $column['table_name'];

    if ($table !== $currentTable) {
        $currentTable = $table;
        echo "\n--- {$table} ---\n";
    }

    echo $column['column_name'] . ' : ' . $column['data_type'] . "\n";
}

echo '</pre>';

Открой:

/local/sitebuilder/migrations/check_sitebuilder_tables.php

И пришли вывод, особенно список таблиц и таблицу, где есть поля примерно:

id
site_id
parent_id
title / name
slug


---

2. Почему это нужно

Для прав на подстраницы нам обязательно надо знать родителя страницы:

page_id → parent_id → parent_id → ...

Сейчас код ищет это здесь:

FROM sitebuilder.pages
WHERE site_id = :site_id
  AND id = :page_id

Но у тебя таблица страниц называется иначе. Например может быть:

sitebuilder.site_pages
sitebuilder.page
sitebuilder.pages_json
sitebuilder.sb_pages

Нужно подставить правильное название.


---

3. Пока можно проверить pageAccess.save

Если ты запускаешь тест под админом Битрикса, pageAccess.save уже может пройти, потому что админ обходится без проверки sitebuilder.pages.

Попробуй ещё раз консольный тест выдачи прав. Если вернётся ok: true, значит таблица page_access работает.

Но перед подключением прав к дереву страниц всё равно надо найти реальную таблицу страниц.