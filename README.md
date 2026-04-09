Зайти нужно туда, где у тебя лежит новая папка проекта под /local.

Если ты создал её как:

/local/sitebuilder/

то базовые ссылки будут такие:

https://ТВОЙ_ДОМЕН/local/sitebuilder/
https://ТВОЙ_ДОМЕН/local/sitebuilder/api.php

Проверять можно так.

1. Проверка, что PHP-точка API вообще открывается

Открой:

https://ТВОЙ_ДОМЕН/local/sitebuilder/api.php

Если просто открыть в браузере GET-запросом, ты, скорее всего, увидишь ошибку вроде:

METHOD_NOT_ALLOWED или

NOT_AUTHORIZED


Это нормально, потому что API ждёт:

авторизованного пользователя

POST

sessid



---

2. Самая правильная проверка API

Сделай временную тестовую страницу, например:

/local/sitebuilder/test_api.php

И зайди по ссылке:

https://ТВОЙ_ДОМЕН/local/sitebuilder/test_api.php

Вот готовый файл для проверки:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
global $USER;
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder API Test</title>
</head>
<body>
    <h1>Проверка API</h1>

    <button id="pingBtn">Ping</button>
    <button id="siteListBtn">Site list</button>

    <pre id="out" style="white-space: pre-wrap; background:#f5f5f5; padding:16px; border:1px solid #ccc;"></pre>

    <script>
        function callApi(data) {
            BX.ajax({
                url: '/local/sitebuilder/api.php',
                method: 'POST',
                data: Object.assign({
                    sessid: BX.bitrix_sessid()
                }, data),
                dataType: 'json',
                onsuccess: function (res) {
                    document.getElementById('out').textContent = JSON.stringify(res, null, 2);
                },
                onfailure: function (err) {
                    document.getElementById('out').textContent = 'AJAX ERROR: ' + JSON.stringify(err, null, 2);
                }
            });
        }

        document.getElementById('pingBtn').onclick = function () {
            callApi({ action: 'ping' });
        };

        document.getElementById('siteListBtn').onclick = function () {
            callApi({ action: 'site.list' });
        };
    </script>
</body>
</html>

Но лучше сразу с подключением BX, иначе BX.ajax не сработает. Вот правильная версия:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
\Bitrix\Main\UI\Extension::load('main.core');
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder API Test</title>
</head>
<body>
    <h1>Проверка API</h1>

    <button id="pingBtn">Ping</button>
    <button id="siteListBtn">Site list</button>

    <pre id="out" style="white-space: pre-wrap; background:#f5f5f5; padding:16px; border:1px solid #ccc;"></pre>

    <script>
        function callApi(data) {
            BX.ajax({
                url: '/local/sitebuilder/api.php',
                method: 'POST',
                data: Object.assign({
                    sessid: BX.bitrix_sessid()
                }, data),
                dataType: 'json',
                onsuccess: function (res) {
                    document.getElementById('out').textContent = JSON.stringify(res, null, 2);
                },
                onfailure: function (err) {
                    document.getElementById('out').textContent = 'AJAX ERROR: ' + JSON.stringify(err, null, 2);
                }
            });
        }

        document.getElementById('pingBtn').onclick = function () {
            callApi({ action: 'ping' });
        };

        document.getElementById('siteListBtn').onclick = function () {
            callApi({ action: 'site.list' });
        };
    </script>
</body>
</html>


---

3. Ссылка для проверки

Если создашь этот файл, заходить надо сюда:

https://ТВОЙ_ДОМЕН/local/sitebuilder/test_api.php


---

4. Что должно быть

При нажатии Ping должен прийти JSON примерно такой:

{
  "ok": true,
  "time": "2026-04-09T...",
  "userId": 1,
  "login": "admin"
}

При нажатии Site list:

либо список сайтов

либо пустой массив

либо ACCESS_DENIED, если нет доступа

либо NOT_AUTHORIZED, если не залогинен



---

5. Если хочешь проверить без страницы

Можно через Postman, но нужно передать:

авторизационные cookie Bitrix

POST

sessid

action


Поэтому через test_api.php проще всего.


---

6. Если новая папка называется не sitebuilder

Тогда ссылка будет по её имени, например если папка:

/local/sitebuilder_new/

то проверка:

https://ТВОЙ_ДОМЕН/local/sitebuilder_new/test_api.php
https://ТВОЙ_ДОМЕН/local/sitebuilder_new/api.php

Если хочешь, следующим сообщением я могу сразу дать тебе готовый test_api.php уже с кнопками для:

site.create

page.list

page.create

block.list

menu.list


чтобы ты одним экраном проверял весь API.