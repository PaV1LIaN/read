Это почти наверняка не ошибка JS, а ошибка PHP/пути в api.php, из-за которой AJAX получает невалидный ответ или 500.

"detail": "status" у BX.ajax в таком случае обычно означает:

неправильный URL API

500 на сервере

fatal error в PHP

в ответ пришёл не JSON


И у тебя есть очень сильная подсказка: ты писал, что создал новую чистую папку под проект.
А в коде у нас много мест с жёстким путём:

/local/sitebuilder/

Если новая папка называется не sitebuilder, то bootstrap.php сейчас, скорее всего, подключает несуществующие файлы и падает с 500.


---

Что проверить первым делом

1. Проверь имя папки проекта

Если папка называется, например:

/local/sitebuilder_new/

то у тебя сейчас точно сломаны вот эти строки в api/bootstrap.php:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/json.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/response.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/access.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/helpers.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/disk.php';

Их надо заменить на реальный путь к твоей новой папке.

Например, если папка /local/sitebuilder_new, то так:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder_new/lib/json.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder_new/lib/response.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder_new/lib/access.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder_new/lib/helpers.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder_new/lib/disk.php';


---

2. Проверь $basePath в страницах

В index.php, menu.php, test_api.php и других страницах у нас стоит:

$basePath = '/local/sitebuilder';

Если папка называется иначе, тоже меняй.

Например:

$basePath = '/local/sitebuilder_new';


---

Самая вероятная причина у тебя

С высокой вероятностью проблема именно тут:

/local/ТВОЯ_ПАПКА/api/bootstrap.php

Замени файл целиком на этот вариант, где путь задаётся автоматически от текущей папки, без хардкода.

<?php

define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;
use Bitrix\Disk\Storage;
use Bitrix\Disk\Folder;
use Bitrix\Disk\File;
use Bitrix\Disk\Driver;

global $USER;

header('Content-Type: application/json; charset=UTF-8');

if (!$USER->IsAuthorized()) {
    http_response_code(401);
    echo json_encode(['ok' => false, 'error' => 'NOT_AUTHORIZED'], JSON_UNESCAPED_UNICODE);
    exit;
}

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    http_response_code(405);
    echo json_encode(['ok' => false, 'error' => 'METHOD_NOT_ALLOWED'], JSON_UNESCAPED_UNICODE);
    exit;
}

if (!check_bitrix_sessid()) {
    http_response_code(403);
    echo json_encode(['ok' => false, 'error' => 'BAD_SESSID'], JSON_UNESCAPED_UNICODE);
    exit;
}

$projectRoot = dirname(__DIR__);

require_once $projectRoot . '/lib/json.php';
require_once $projectRoot . '/lib/response.php';
require_once $projectRoot . '/lib/access.php';
require_once $projectRoot . '/lib/helpers.php';
require_once $projectRoot . '/lib/disk.php';

Вот это уже правильнее, потому что не зависит от имени папки.


---

И сразу исправь страницы так же

Чтобы не держать хардкод, в index.php, menu.php и дальше лучше ставить basePath автоматически.

В index.php

вместо:

$basePath = '/local/sitebuilder';

поставь:

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');

В menu.php

вместо:

$basePath = '/local/sitebuilder';

поставь то же самое:

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');

Это автоматически даст:

/local/sitebuilder

или /local/sitebuilder_new

или любой другой путь



---

Как быстро понять, что именно сейчас ломается

Открой в браузере напрямую:

/local/ТВОЯ_ПАПКА/api.php

Если всё подключается правильно, ты должен увидеть JSON примерно такой:

{"ok":false,"error":"METHOD_NOT_ALLOWED"}

Потому что ты откроешь GET-запросом.

Если вместо этого:

белая страница

500

HTML с ошибкой

warning/fatal


значит проблема точно в PHP внутри api.php / bootstrap.php.


---

Самый полезный быстрый тест

Временно замени /local/ТВОЯ_ПАПКА/api.php на:

<?php

ini_set('display_errors', '1');
error_reporting(E_ALL);

require_once __DIR__ . '/api/index.php';

Если проблема в fatal error, он покажется прямо в ответе.

Потом это вернём обратно.


---

Что я рекомендую сделать прямо сейчас

1. В api/bootstrap.php убрать хардкод и поставить:



$projectRoot = dirname(__DIR__);
require_once $projectRoot . '/lib/json.php';
require_once $projectRoot . '/lib/response.php';
require_once $projectRoot . '/lib/access.php';
require_once $projectRoot . '/lib/helpers.php';
require_once $projectRoot . '/lib/disk.php';

2. В index.php и menu.php заменить $basePath на:



$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');

3. Открыть напрямую:



/local/ТВОЯ_ПАПКА/api.php

4. Если не открылся JSON METHOD_NOT_ALLOWED, временно включить display_errors в api.php




---

Почему я почти уверен, что это именно это

Потому что:

кнопки нажимаются

BX.ajax уже работает

ошибка одинаковая и в index.php, и в menu.php

ты создавал новую чистую папку

у нас в нескольких файлах был жёсткий путь /local/sitebuilder


Это типичный сценарий: фронт живой, а API падает на require_once.

Напиши, что показывает прямой заход на api.php, или пришли точное имя новой папки, и я скажу, что ещё поправить.