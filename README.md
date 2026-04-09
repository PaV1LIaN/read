Да, ты прав по обоим пунктам.

1. Шапку/подвал Bitrix подключать не надо, если это отдельный интерфейс конструктора.


2. В include.php здесь вообще не нужна никакая логика. Я дал лишний мусор, из-за этого проще всего просто сделать пустой безопасный include.php.



Ниже даю исправленные готовые файлы.


---

1. /local/modules/its.sitebuilder/include.php

Полностью замени на это:

<?php

Да, просто так.
Для старта модуля этого достаточно.


---

2. /local/sitebuilder/index.php

Тоже замени полностью, без подключения шаблона Bitrix:

<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('NOT_CHECK_PERMISSIONS', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;
use Bitrix\Main\UI\Extension;

global $USER;

if (!$USER || !$USER->IsAuthorized()) {
    http_response_code(401);
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>Site Builder</title>
    </head>
    <body>
        <div style="padding:20px;font-family:Arial,sans-serif;color:#b00020;">
            Доступ запрещён. Пользователь не авторизован.
        </div>
    </body>
    </html>
    <?php
    exit;
}

if (!Loader::includeModule('its.sitebuilder')) {
    http_response_code(500);
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>Site Builder</title>
    </head>
    <body>
        <div style="padding:20px;font-family:Arial,sans-serif;color:#b00020;font-weight:bold;">
            Модуль its.sitebuilder не установлен или не подключается.
        </div>
    </body>
    </html>
    <?php
    exit;
}

Extension::load('main.core');
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ITS Site Builder</title>
</head>
<body style="margin:0;background:#f5f7fb;font-family:Arial,sans-serif;">
    <div style="padding:20px;">
        <h1 style="margin-top:0;">ITS Site Builder</h1>

        <p>
            Это тестовая страница нового модуля без шаблона Bitrix.
        </p>

        <div style="margin:20px 0;">
            <button id="sb-ping-btn" style="padding:10px 16px;cursor:pointer;">
                Проверить контроллер ping
            </button>

            <button id="sb-demo-list-btn" style="padding:10px 16px;cursor:pointer;margin-left:10px;">
                Проверить demoList
            </button>
        </div>

        <pre id="sb-result" style="background:#fff;border:1px solid #d0d7de;padding:16px;white-space:pre-wrap;min-height:180px;">Нажми кнопку для проверки.</pre>
    </div>

    <script>
    BX.ready(function () {
        var resultNode = document.getElementById('sb-result');
        var pingBtn = document.getElementById('sb-ping-btn');
        var listBtn = document.getElementById('sb-demo-list-btn');

        function printResult(title, data) {
            resultNode.textContent = title + "\n\n" + JSON.stringify(data, null, 2);
        }

        pingBtn.addEventListener('click', function () {
            printResult('Запрос...', { action: 'its:sitebuilder.Site.ping' });

            BX.ajax.runAction('its:sitebuilder.Site.ping')
                .then(function (response) {
                    printResult('Успех', response);
                })
                .catch(function (error) {
                    printResult('Ошибка', error);
                });
        });

        listBtn.addEventListener('click', function () {
            printResult('Запрос...', { action: 'its:sitebuilder.Site.demoList' });

            BX.ajax.runAction('its:sitebuilder.Site.demoList')
                .then(function (response) {
                    printResult('Успех', response);
                })
                .catch(function (error) {
                    printResult('Ошибка', error);
                });
        });
    });
    </script>
</body>
</html>


---

3. /local/modules/its.sitebuilder/.settings.php

На всякий случай ещё раз привожу целиком:

<?php

return [
    'controllers' => [
        'value' => [
            'defaultNamespace' => '\\Its\\Sitebuilder\\Controller',
        ],
        'readonly' => true,
    ],
];


---

4. /local/modules/its.sitebuilder/lib/controller/site.php

И тоже лучше сразу поставить вот этот вариант:

<?php

namespace Its\Sitebuilder\Controller;

use Bitrix\Main\Engine\Controller;

class Site extends Controller
{
    public function pingAction(): array
    {
        global $USER;

        return [
            'ok' => true,
            'module' => 'its.sitebuilder',
            'controller' => static::class,
            'userId' => is_object($USER) ? (int)$USER->GetID() : 0,
            'time' => date('c'),
        ];
    }

    public function demoListAction(): array
    {
        return [
            'items' => [
                [
                    'id' => 1,
                    'name' => 'Демо сайт 1',
                    'slug' => 'demo-1',
                ],
                [
                    'id' => 2,
                    'name' => 'Демо сайт 2',
                    'slug' => 'demo-2',
                ],
            ],
        ];
    }

    public function configureActions(): array
    {
        return [
            'ping' => [
                'prefilters' => [],
            ],
            'demoList' => [
                'prefilters' => [],
            ],
        ];
    }
}


---

Почему упало

Причина была в том, что для include.php я дал ненужную конструкцию.
В модуле на старте нам сейчас вообще не нужен никакой код в include.php, поэтому safest-вариант — оставить просто:

<?php


---

Что сделать сейчас

1. Замени include.php на пустой.


2. Замени index.php на вариант без шаблона Bitrix.


3. Открой:



/local/sitebuilder/index.php

4. Проверь две кнопки.



Если после этого будет ошибка уже не parse, а например контроллер не найдётся — следующим сообщением я дам тебе сразу полный рабочий вариант с sitebuilder.php вместо D7 controller, если твоя коробка капризно работает с Engine\Controller.