Можно. И это даже лучше для первого прогона.

Самый простой путь — протестировать компонент как отдельную страницу вне sitebuilder, подставив вручную siteId/pageId/blockId.


---

Что сделать

1. Создай тестовый файл

Например:

/local/sitebuilder/components/disk/test.php


---

2. Вставь туда такой код

<?php
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/components/disk/class.php';

global $USER;

if (!$USER->IsAuthorized()) {
    die('Нужно авторизоваться в Битрикс');
}

$siteId = 1;   // подставь существующий site_id
$pageId = 1;   // подставь существующий page_id
$blockId = 1;  // подставь существующий block_id типа disk
$currentUserId = (int)$USER->GetID();
?>
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Тест компонента Disk</title>
    <link rel="stylesheet" href="/local/sitebuilder/components/disk/styles.css">
</head>
<body style="margin:0; padding:24px; background:#f5f7fb;">
    <div style="max-width:1200px; margin:0 auto;">
        <?php
        $component = new SitebuilderDiskComponent([
            'SITE_ID' => $siteId,
            'PAGE_ID' => $pageId,
            'BLOCK_ID' => $blockId,
            'CURRENT_USER_ID' => $currentUserId,
        ]);
        $component->execute();
        ?>
    </div>

    <script src="/bitrix/js/main/core/core.js"></script>
    <script src="/local/sitebuilder/components/disk/script.js"></script>
</body>
</html>


---

Что нужно подготовить до теста

1. В БД должен существовать сайт

В таблице sitebuilder_site должна быть запись, например:

id = 1



---

2. В БД должна существовать страница этого сайта

В таблице sitebuilder_page:

id = 1

site_id = 1



---

3. В БД должен существовать блок типа disk

В таблице sitebuilder_block:

id = 1

site_id = 1

page_id = 1

type = 'disk'

is_active = 1



---

4. Для пользователя должна быть роль на сайте

В таблице sitebuilder_site_user_access:

site_id = 1

user_id = ТВОЙ_ID

role_code = 'site_admin'



---

Как быстро создать тестовые данные вручную

Если записей еще нет, можно вставить их SQL.

Сайт

INSERT INTO sitebuilder_site (id, name, code, root_disk_folder_id, settings_json, created_at, updated_at)
VALUES (1, 'Тестовый сайт', 'test-site', NULL, '{}', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

Страница

INSERT INTO sitebuilder_page (id, site_id, title, slug, sort, created_at, updated_at)
VALUES (1, 1, 'Главная', 'index', 100, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

Блок

INSERT INTO sitebuilder_block (id, site_id, page_id, type, sort, settings_json, is_active, created_by, created_at, updated_at)
VALUES (1, 1, 1, 'disk', 100, '{}', 1, 1, CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

Роль пользователю

INSERT INTO sitebuilder_site_user_access (site_id, user_id, role_code, created_at, updated_at)
VALUES (1, 1, 'site_admin', CURRENT_TIMESTAMP, CURRENT_TIMESTAMP);

Если id=1 уже заняты, подставь другие и такие же значения укажи в test.php.


---

Как открыть тест

Открой в браузере:

https://твой-домен/local/sitebuilder/components/disk/test.php


---

Что должно произойти

Сценарий 1. Root еще не создан

Ты увидишь состояние:

“Для блока не настроена корневая папка”

кнопки:

“Создать корень сайта”

“Создать папку блока”



Это уже хороший знак: компонент живой.


---

Сценарий 2. Root создался

После нажатия:

создастся папка в Bitrix Disk

у сайта или блока сохранится root_folder_id

блок загрузит содержимое папки



---

Сценарий 3. Проверка функций

Потом можно проверить:

создание папки

загрузку файла

открытие вложенной папки

поиск

переименование

удаление

настройки блока



---

Как проверить по частям, если что-то не работает

1. Проверить API отдельно

Можно дергать экшены прямо через браузер/DevTools/Postman.

Например, открыть страницу теста, потом в консоли браузера:

fetch('/local/sitebuilder/components/disk/api.php?action=getSettings', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    siteId: 1,
    pageId: 1,
    blockId: 1,
    sessid: BX.bitrix_sessid()
  })
}).then(r => r.json()).then(console.log)

Если это работает — серверная часть уже живая.


---

2. Проверить resolveRoot

fetch('/local/sitebuilder/components/disk/api.php?action=resolveRoot', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    siteId: 1,
    pageId: 1,
    blockId: 1,
    sessid: BX.bitrix_sessid()
  })
}).then(r => r.json()).then(console.log)


---

3. Проверить создание корня сайта

fetch('/local/sitebuilder/components/disk/api.php?action=initSiteRoot', {
  method: 'POST',
  headers: {'Content-Type': 'application/json'},
  body: JSON.stringify({
    siteId: 1,
    sessid: BX.bitrix_sessid()
  })
}).then(r => r.json()).then(console.log)


---

Если хочешь вообще без блока в БД

Теоретически можно и так, но твоя текущая архитектура специально завязана на blockId, поэтому без записи в sitebuilder_block тестировать неудобно.

Минимум для теста нужен именно такой набор:

один site

одна page

один block типа disk

одна роль для пользователя


Это уже не sitebuilder UI, но уже честный тест компонента.


---

Самый удобный способ теста

Лучший вариант сейчас:

1. создать test.php


2. создать 1 тестовый сайт/страницу/блок в БД


3. выдать себе роль site_admin


4. открыть /local/sitebuilder/components/disk/test.php



Так ты проверишь почти весь компонент без интеграции в editor/sitebuilder.

Если хочешь, следующим сообщением я могу прислать готовый test.php + SQL-вставки для тестовых данных одним куском.