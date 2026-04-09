Сейчас уже ясно одно: запрос доходит до /local/sitebuilder/api.php, но сервер отдаёт 501, значит внутри PHP всё ещё исполняется старый код или не тот файл.

Раз api.php с жёсткой заменой не изменил поведение, делаем самый прямой тест: вообще убираем весь проект из цепочки.

Замени /local/sitebuilder/api.php целиком на это

<?php
header('Content-Type: application/json; charset=UTF-8');
http_response_code(200);

echo json_encode([
    'ok' => true,
    'step' => 'API_PHP_DIRECT',
    'file' => __FILE__,
    'time' => date('c'),
], JSON_UNESCAPED_UNICODE);

exit;

Потом открой напрямую в браузере

https://portal24.itsnn.ru/local/sitebuilder/api.php

Ты должен увидеть строго это

Что-то вроде:

{"ok":true,"step":"API_PHP_DIRECT","file":"/srv/bx/docroot/local/sitebuilder/api.php","time":"..."}


---

Если ты не увидишь этот JSON

Тогда проблема не в index.php, не в handlers и не в opcache.

Тогда остаются только такие варианты:

ты редактируешь не тот физический файл

nginx/angie/PHP-FPM отдаёт другой docroot

есть rewrite/alias/symlink

/local/sitebuilder/api.php на диске не совпадает с тем, что обслуживает веб



---

Если этот JSON откроется напрямую

Тогда нажми Ping в test_api.php.

Если Ping всё равно даст 501

Значит test_api.php стучится не в тот URL.

Тогда в test_api.php временно захардкодь URL вот так:

url: 'https://portal24.itsnn.ru/local/sitebuilder/api.php',

И везде, где сейчас:

url: '/local/sitebuilder/api.php',

замени на полный URL.


---

Следующий шаг после проверки

Если прямой заход на api.php покажет не тот JSON, тогда на сервере надо проверить, что реально обслуживается:

readlink -f /srv/bx/docroot/local/sitebuilder/api.php
ls -l /srv/bx/docroot/local/sitebuilder/api.php

Но сначала нужен результат прямого открытия https://portal24.itsnn.ru/local/sitebuilder/api.php.

Пришли ровно то, что откроется в браузере по этой ссылке.