Отлично. Это очень хороший результат.

Теперь точно известно:

test_api.php ходит в правильный /local/sitebuilder/api.php

api.php реально меняется

проблема была внутри цепочки api/index.php / handlers, а не в AJAX


То есть мы наконец сняли всю неопределённость.


---

Что делать дальше

Сейчас нужно аккуратно вернуть api.php к нормальной работе, но не сразу весь проект, а поэтапно.

Шаг 1. Верни /local/sitebuilder/api.php

Сделай так:

<?php
ini_set('display_errors', '1');
error_reporting(E_ALL);

require_once __DIR__ . '/api/index.php';


---

Шаг 2. Поставь минимальный живой api/index.php

Не весь роутер сразу. Сначала только ping и site.list.

/local/sitebuilder/api/index.php

<?php

require_once __DIR__ . '/bootstrap.php';

$action = (string)($_POST['action'] ?? '');

if ($action === 'ping') {
    require __DIR__ . '/handlers/common.php';
    exit;
}

if ($action === 'site.list') {
    require __DIR__ . '/handlers/site.php';
    exit;
}

sb_json_error('UNKNOWN_ACTION', 400, [
    'action' => $action,
    'file' => __FILE__,
]);


---

Шаг 3. Поставь минимальный живой common.php

/local/sitebuilder/api/handlers/common.php

<?php

global $USER;

if ($action === 'ping') {
    sb_json_ok([
        'pong' => true,
        'handler' => 'common',
        'file' => __FILE__,
        'userId' => (int)$USER->GetID(),
        'login' => (string)$USER->GetLogin(),
        'time' => date('c'),
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'common',
    'action' => $action,
    'file' => __FILE__,
]);


---

Шаг 4. Поставь минимальный живой site.php

Чтобы исключить любую путаницу, сначала не полный CRUD, а только site.list.

/local/sitebuilder/api/handlers/site.php

<?php

if ($action === 'site.list') {
    $sites = sb_read_sites();
    $myCode = sb_user_access_code();

    $access = sb_read_access();
    $allowedSiteIds = [];

    foreach ($access as $r) {
        if ((string)($r['accessCode'] ?? '') === $myCode) {
            $sid = (int)($r['siteId'] ?? 0);
            if ($sid > 0) {
                $allowedSiteIds[$sid] = true;
            }
        }
    }

    $sites = array_values(array_filter($sites, static function ($s) use ($allowedSiteIds) {
        return isset($allowedSiteIds[(int)($s['id'] ?? 0)]);
    }));

    usort($sites, static function ($a, $b) {
        return (int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0);
    });

    sb_json_ok([
        'sites' => $sites,
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

sb_json_error('NOT_MOVED_YET', 501, [
    'handler' => 'site',
    'action' => $action,
    'file' => __FILE__,
]);


---

Что проверить теперь

1. В test_api.php нажми Ping

Ожидаемый ответ:

{
  "ok": true,
  "pong": true,
  "handler": "common",
  "file": "/srv/bx/docroot/local/sitebuilder/api/handlers/common.php"
}

2. Нажми Site list

Ожидаемый ответ:

{
  "ok": true,
  "sites": [],
  "handler": "site",
  "file": "/srv/bx/docroot/local/sitebuilder/api/handlers/site.php"
}

или не пустой список.


---

Почему делаем именно так

Потому что раньше у тебя было слишком много переменных сразу:

весь роутер

все handlers

possible путаница файлов


Сейчас мы поднимаем API как лестницу:

1. api.php


2. api/index.php


3. common.php


4. site.php


5. потом постепенно добавляем остальные actions




---

Если это заработает

Тогда следующим сообщением я дам тебе уже правильный расширенный site.php:

site.get

site.create

site.delete

site.setHome


А после него — вернём page.


---

Важный момент

Пока не возвращай:

access.php

page.php

block.php

остальные handlers в роутер


Сначала нужно добиться, чтобы:

ping шёл в common.php

site.list шёл в site.php


Это будет контрольная точка.

Пришли мне два ответа:

что вернул Ping

что вернул Site list


И после этого пойдём дальше без путаницы.