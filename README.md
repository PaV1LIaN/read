Тогда делаем точечную диагностику без гадания.

Сейчас задача одна: понять, какой файл реально подключается на site.list.

Шаг 1. Временно замени /local/ТВОЯ_ПАПКА/api/index.php на этот диагностический вариант

<?php

require_once __DIR__ . '/bootstrap.php';

$action = (string)($_POST['action'] ?? '');

$map = [
    'ping' => __DIR__ . '/handlers/common.php',

    'site.list' => __DIR__ . '/handlers/site.php',
    'site.get' => __DIR__ . '/handlers/site.php',
    'site.create' => __DIR__ . '/handlers/site.php',
    'site.update' => __DIR__ . '/handlers/site.php',
    'site.delete' => __DIR__ . '/handlers/site.php',
    'site.setHome' => __DIR__ . '/handlers/site.php',

    'page.list' => __DIR__ . '/handlers/page.php',
    'page.create' => __DIR__ . '/handlers/page.php',
    'page.delete' => __DIR__ . '/handlers/page.php',
    'page.duplicate' => __DIR__ . '/handlers/page.php',
    'page.updateMeta' => __DIR__ . '/handlers/page.php',
    'page.setStatus' => __DIR__ . '/handlers/page.php',
    'page.setParent' => __DIR__ . '/handlers/page.php',
    'page.move' => __DIR__ . '/handlers/page.php',

    'block.list' => __DIR__ . '/handlers/block.php',
    'block.create' => __DIR__ . '/handlers/block.php',
    'block.update' => __DIR__ . '/handlers/block.php',
    'block.delete' => __DIR__ . '/handlers/block.php',
    'block.duplicate' => __DIR__ . '/handlers/block.php',
    'block.move' => __DIR__ . '/handlers/block.php',
    'block.reorder' => __DIR__ . '/handlers/block.php',

    'access.list' => __DIR__ . '/handlers/access.php',
    'access.set' => __DIR__ . '/handlers/access.php',
    'access.delete' => __DIR__ . '/handlers/access.php',

    'file.list' => __DIR__ . '/handlers/file.php',
    'file.upload' => __DIR__ . '/handlers/file.php',
    'file.delete' => __DIR__ . '/handlers/file.php',

    'menu.list' => __DIR__ . '/handlers/menu.php',
    'menu.create' => __DIR__ . '/handlers/menu.php',
    'menu.update' => __DIR__ . '/handlers/menu.php',
    'menu.delete' => __DIR__ . '/handlers/menu.php',
    'menu.setTop' => __DIR__ . '/handlers/menu.php',
    'menu.item.add' => __DIR__ . '/handlers/menu.php',
    'menu.item.update' => __DIR__ . '/handlers/menu.php',
    'menu.item.delete' => __DIR__ . '/handlers/menu.php',
    'menu.item.move' => __DIR__ . '/handlers/menu.php',

    'template.list' => __DIR__ . '/handlers/template.php',
    'template.createFromPage' => __DIR__ . '/handlers/template.php',
    'template.applyToPage' => __DIR__ . '/handlers/template.php',
    'template.rename' => __DIR__ . '/handlers/template.php',
    'template.delete' => __DIR__ . '/handlers/template.php',

    'layout.get' => __DIR__ . '/handlers/layout.php',
    'layout.updateSettings' => __DIR__ . '/handlers/layout.php',
    'layout.block.list' => __DIR__ . '/handlers/layout.php',
    'layout.block.create' => __DIR__ . '/handlers/layout.php',
    'layout.block.update' => __DIR__ . '/handlers/layout.php',
    'layout.block.delete' => __DIR__ . '/handlers/layout.php',
    'layout.block.move' => __DIR__ . '/handlers/layout.php',
];

if (!isset($map[$action])) {
    sb_json_error('UNKNOWN_ACTION', 400, [
        'action' => $action,
        'indexFile' => __FILE__,
    ]);
}

$handler = $map[$action];

sb_json_response([
    'ok' => false,
    'debug' => true,
    'action' => $action,
    'indexFile' => __FILE__,
    'handlerFile' => $handler,
    'handlerExists' => file_exists($handler),
    'siteHandlerMd5' => file_exists(__DIR__ . '/handlers/site.php') ? md5_file(__DIR__ . '/handlers/site.php') : null,
    'accessHandlerMd5' => file_exists(__DIR__ . '/handlers/access.php') ? md5_file(__DIR__ . '/handlers/access.php') : null,
], 200);

Теперь нажми в test_api.php кнопку Site list.

Что ты должен получить

Вместо старой ошибки должен прийти JSON примерно такого вида:

{
  "ok": false,
  "debug": true,
  "action": "site.list",
  "indexFile": ".../api/index.php",
  "handlerFile": ".../api/handlers/site.php",
  "handlerExists": true,
  "siteHandlerMd5": "...",
  "accessHandlerMd5": "..."
}

Если придёт не это

Тогда запрос вообще идёт не в тот api/index.php, и значит используется другой проект/другая папка/старый файл.


---

Шаг 2. Временно замени /local/ТВОЯ_ПАПКА/api.php на диагностический

<?php

ini_set('display_errors', '1');
error_reporting(E_ALL);

header('Content-Type: application/json; charset=UTF-8');

echo json_encode([
    'ok' => false,
    'debug' => true,
    'apiFile' => __FILE__,
    'dir' => __DIR__,
]);
exit;

Потом снова нажми любую кнопку.

Что важно увидеть

Какой именно api.php реально отвечает.

Если в ответе будет путь не из той папки, значит фронт стучится в другой проект.


---

Шаг 3. Проверь путь в index.php и test_api.php

В этих файлах должно быть:

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');

И в JS должен строиться URL так:

var API_URL = BASE_PATH + '/api.php';

А не хардкодом на старую папку.


---

Почему я сейчас почти уверен в этом

Потому что ответ:

{"ok":false,"error":"NOT_MOVED_YET","handler":"access","action":"site.list"}

слишком “точный”. Это не случайная серверная ошибка. Это означает, что:

либо реально подключается access.php,

либо подключается другой старый набор файлов, где роутинг настроен иначе,

либо срабатывает старый opcache/не та папка.



---

Самый быстрый путь сейчас

Сделай шаг 1 с диагностическим api/index.php и пришли мне ответ JSON целиком.
По нему сразу станет видно:

тот ли index.php

тот ли handler-файл

существует ли site.php

не упираемся ли в другую папку/кэш.