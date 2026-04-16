Отлично. Тогда следующий шаг — подключить блок в sitebuilder, чтобы он реально появился в системе.

Ниже даю, что делать дальше после таблиц и файлов компонента.


---

1. Зарегистрировать тип блока disk

Найди место, где у тебя хранится реестр блоков конструктора.

Это может быть что-то вроде:

/local/sitebuilder/config/blocks.php

/local/sitebuilder/lib/blocks.php

/local/sitebuilder/data/blocks.php

или массив в editor.php


Туда нужно добавить блок:

'disk' => [
    'type' => 'disk',
    'name' => 'Диск',
    'icon' => 'disk',
    'category' => 'content',
    'component_path' => '/local/sitebuilder/components/disk/',
    'supports_settings' => true,
    'supports_permissions' => true,
    'is_active' => true,
],

Если у тебя блоки хранятся массивом без ключа, добавь как обычный элемент:

[
    'type' => 'disk',
    'name' => 'Диск',
    'icon' => 'disk',
    'category' => 'content',
    'component_path' => '/local/sitebuilder/components/disk/',
    'supports_settings' => true,
    'supports_permissions' => true,
    'is_active' => true,
],


---

2. Добавить создание блока disk через API/обработчик блока

У тебя уже есть логика создания блоков страницы.
Там, где создается новый блок, нужно добавить ветку для type = disk.

Если создание идет через handler API

Например, в чем-то вроде:

/local/sitebuilder/api/handlers/block.php
/local/sitebuilder/api/handlers/page.php
/local/sitebuilder/api/handlers/site.php

Нужна логика:

case 'disk':
    $blockId = BlockRepository::create([
        'site_id' => $siteId,
        'page_id' => $pageId,
        'type' => 'disk',
        'sort' => $sort,
        'settings_json' => '{}',
        'is_active' => 1,
        'created_by' => $currentUserId,
    ]);

    DiskSettingsRepository::createDefault([
        'block_id' => $blockId,
        'site_id' => $siteId,
        'page_id' => $pageId,
        'created_by' => $currentUserId,
    ]);
    break;


---

3. Если у тебя есть общий helper создания блока

Лучше вынести в отдельную функцию:

function sitebuilder_create_disk_block(int $siteId, int $pageId, int $currentUserId, int $sort = 500): int
{
    $blockId = BlockRepository::create([
        'site_id' => $siteId,
        'page_id' => $pageId,
        'type' => 'disk',
        'sort' => $sort,
        'settings_json' => '{}',
        'is_active' => 1,
        'created_by' => $currentUserId,
    ]);

    DiskSettingsRepository::createDefault([
        'block_id' => $blockId,
        'site_id' => $siteId,
        'page_id' => $pageId,
        'created_by' => $currentUserId,
    ]);

    return $blockId;
}


---

4. Подключить рендер блока на странице

Теперь нужно, чтобы при выводе страницы блок disk реально рендерился.

Найди место, где у тебя идет цикл по блокам страницы.
Это может быть:

editor.php

public.php

renderer.php

page renderer

или что-то внутри /local/sitebuilder/


Примерно там есть что-то вроде:

foreach ($blocks as $block) {
    switch ($block['type']) {
        case 'text':
            // ...
            break;

        case 'image':
            // ...
            break;
    }
}

Добавь туда:

case 'disk':
    require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/class.php';

    $component = new SitebuilderDiskComponent([
        'SITE_ID' => (int)$siteId,
        'PAGE_ID' => (int)$pageId,
        'BLOCK_ID' => (int)$block['id'],
        'CURRENT_USER_ID' => (int)$USER->GetID(),
    ]);

    $component->execute();
    break;


---

5. Подключить styles.css и script.js

Чтобы блок работал, на странице должны быть подключены:

<link rel="stylesheet" href="/local/sitebuilder/components/disk/styles.css">
<script src="/local/sitebuilder/components/disk/script.js"></script>

Лучше подключать один раз на страницу, а не при каждом блоке.

Вариант

Если у тебя есть общий layout/editor page header, вставь туда:

<?php if ($pageHasDiskBlock): ?>
    <link rel="stylesheet" href="/local/sitebuilder/components/disk/styles.css">
    <script src="/local/sitebuilder/components/disk/script.js"></script>
<?php endif; ?>

Если такой оптимизации нет, на первом этапе можно просто подключить всегда.


---

6. Проверить, что PDO реально подключен

До первого запуска обязательно проверь DiskDb.php.

Сейчас там заглушка:

throw new RuntimeException('DB_CONNECTION_NOT_CONFIGURED');

Ее надо заменить на ваш реальный доступ к БД.

Например, если у тебя есть свой wrapper:

public static function getConnection(): PDO
{
    if (self::$pdo instanceof PDO) {
        return self::$pdo;
    }

    require_once $_SERVER['DOCUMENT_ROOT'] . '/local/php_interface/lib/pg_master.php';

    $pdo = getPdo();
    if (!$pdo instanceof PDO) {
        throw new RuntimeException('DB_CONNECTION_NOT_CONFIGURED');
    }

    self::$pdo = $pdo;
    self::$pdo->setAttribute(PDO::ATTR_ERRMODE, PDO::ERRMODE_EXCEPTION);
    self::$pdo->setAttribute(PDO::ATTR_DEFAULT_FETCH_MODE, PDO::FETCH_ASSOC);

    return self::$pdo;
}

Если у тебя не getPdo(), а другой метод — подставь свой.


---

7. Проверить SQL-таблицы и названия полей

Код ожидает именно такие таблицы:

sitebuilder_site

sitebuilder_page

sitebuilder_block

sitebuilder_disk_settings

sitebuilder_site_user_access


И такие поля:

В sitebuilder_site

id

name

root_disk_folder_id


В sitebuilder_block

id

site_id

page_id

type

sort

settings_json

is_active

created_by


В sitebuilder_disk_settings

block_id

site_id

page_id

title

root_folder_id

view_mode

allow_upload

allow_create_folder

allow_rename

allow_delete

allow_download

show_search

show_breadcrumbs

default_sort

default_sort_direction

allowed_extensions_json

max_file_size

permission_mode

use_site_root_fallback


Если у тебя таблицы или поля называются иначе — надо поправить репозитории.


---

8. Выдать себе роль на сайт

Без этого DiskPermissionService может закрыть доступ.

В sitebuilder_site_user_access у текущего пользователя должна быть роль, например:

INSERT INTO sitebuilder_site_user_access (site_id, user_id, role_code)
VALUES (1, 1, 'site_admin');

Подставь свой site_id и свой user_id.


---

9. Проверить первый сценарий запуска

После подключения сделай такой тест:

Шаги

1. Открой страницу сайта, где добавлен блок disk


2. Если корня нет — должен показаться state no-root


3. Нажми:

Создать корень сайта или

Создать папку блока



4. Открой настройки


5. Сохрани настройки


6. Проверь:

создание папки

загрузку файла

открытие вложенной папки

поиск

скачивание





---

10. Что может сломаться первым делом

Самые вероятные проблемы:

DB_CONNECTION_NOT_CONFIGURED

Значит не подключен реальный PDO в DiskDb.php

BLOCK_CONTEXT_MISMATCH

Значит:

site_id/page_id/block_id не совпадают

или блок в БД не type = 'disk'


BAD_SESSID

Значит на фронте не передается sessid

DISK_STORAGE_NOT_FOUND

Значит Bitrix Disk storage не найден для пользователя

SITE_NOT_FOUND

Значит в sitebuilder_site нет записи с таким id

ACCESS_DENIED

Значит у пользователя нет роли в sitebuilder_site_user_access


---

11. Что лучше сделать прямо сейчас

Сейчас порядок такой:

1.

Подключить DiskDb.php к реальной БД

2.

Зарегистрировать тип блока disk

3.

Добавить создание блока disk

4.

Добавить рендер блока в цикл рендера страницы

5.

Подключить styles.css и script.js

6.

Выдать себе роль site_admin в sitebuilder_site_user_access

7.

Открыть страницу и проверить первый запуск


---

12. Самый полезный следующий шаг

Дальше лучше идти не в теорию, а в точечную интеграцию под твой проект.

Самое полезное сейчас — прислать мне:

файл, где у тебя регистрируются блоки

файл, где у тебя создаются блоки

файл, где у тебя рендерится страница / блоки


И я скажу конкретно, какой кусок и куда вставить под твой sitebuilder.