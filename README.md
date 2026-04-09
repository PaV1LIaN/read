Отлично. Теперь точно известно:

запрос идёт в нужный файл: /srv/bx/docroot/local/sitebuilder/api.php

папка тоже нужная: /srv/bx/docroot/local/sitebuilder


Значит проблема уже сузилась до двух вариантов:

1. api.php подключает не тот api/index.php


2. api/index.php или handler-файлы не те по содержимому


3. реже — упёрлись в opcache, и PHP держит старую версию файлов



Дальше делаем ещё один точный шаг.

Шаг 1. Замени api.php на такой вариант

<?php

ini_set('display_errors', '1');
error_reporting(E_ALL);

require_once __DIR__ . '/api/bootstrap.php';

$indexFile = __DIR__ . '/api/index.php';

echo json_encode([
    'ok' => false,
    'debug' => true,
    'apiFile' => __FILE__,
    'indexFile' => $indexFile,
    'indexExists' => file_exists($indexFile),
    'indexMd5' => file_exists($indexFile) ? md5_file($indexFile) : null,
    'siteHandlerExists' => file_exists(__DIR__ . '/api/handlers/site.php'),
    'siteHandlerMd5' => file_exists(__DIR__ . '/api/handlers/site.php') ? md5_file(__DIR__ . '/api/handlers/site.php') : null,
    'accessHandlerExists' => file_exists(__DIR__ . '/api/handlers/access.php'),
    'accessHandlerMd5' => file_exists(__DIR__ . '/api/handlers/access.php') ? md5_file(__DIR__ . '/api/handlers/access.php') : null,
    'postedAction' => $_POST['action'] ?? null,
], JSON_UNESCAPED_UNICODE);
exit;

Потом снова нажми Site list в тесте.

Что это даст

Мы увидим:

существует ли api/index.php

существуют ли site.php и access.php

и главное — реально ли POST доходит с action=site.list



---

Шаг 2. После этого открой содержимое трёх файлов и проверь их вручную

/local/sitebuilder/api.php

сейчас должен быть либо диагностический файл, либо потом такой:

<?php
require_once __DIR__ . '/api/index.php';

/local/sitebuilder/api/index.php

там должен быть маппинг:

'site.list' => __DIR__ . '/handlers/site.php',

/local/sitebuilder/api/handlers/site.php

там должен быть код с:

if ($action === 'site.list') {

А не handler => access.


---

Шаг 3. Очень важная проверка на opcache

У тебя может держаться старый php-opcache, и тогда ты меняешь файлы, а PHP отдаёт старую версию.

Сделай временный файл:

/local/sitebuilder/opcache_reset.php

<?php
header('Content-Type: text/plain; charset=UTF-8');

if (function_exists('opcache_reset')) {
    var_dump(opcache_reset());
} else {
    echo "opcache_reset unavailable";
}

Открой его в браузере один раз.

Если увидишь:

bool(true)

значит opcache сброшен.

После этого снова проверь test_api.php.


---

Почему я сейчас подозреваю opcache тоже

Потому что ты уже менял:

api.php

api/index.php


А поведение оставалось “старым”.
Такое бывает, если PHP-FPM держит старый закэшированный скрипт.


---

Самый вероятный сценарий сейчас

По текущим данным я бы оценил так:

50% — api/index.php реально не тот по содержимому

35% — opcache держит старую версию

15% — site.php/access.php перепутаны местами



---

Что мне нужно от тебя следующим сообщением

Пришли JSON, который вернёт новый диагностический api.php после нажатия Site list.

И отдельно, если можешь, просто одной строкой:

что показывает opcache_reset.php


После этого я скажу уже не “куда смотреть”, а какой именно файл у тебя сломан.