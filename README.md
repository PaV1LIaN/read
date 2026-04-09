Идём дальше правильно: сначала делаем диагностический, но уже нормальный рабочий шаг.

Сейчас проблема, скорее всего, в одном из трёх мест:

JS-обработчики не цепляются как надо;

BX.ajax.runAction в этой странице не поднялся;

D7 controller не маршрутизируется в твоей коробке так, как ожидается.


Чтобы не гадать, делаем надёжный промежуточный вариант:
убираем зависимость от runAction и проверяем модуль через обычный ajax-endpoint.

Это не финальная архитектура, но это правильный технический шаг, потому что:

страница уже начнёт реально ходить в новый модуль;

мы не будем упираться в нюансы Engine\Controller;

дальше этот же endpoint можно заменить на нормальный front controller.



---

Что делаем сейчас

Добавляем:

общий bootstrap модуля;

простой ajax endpoint;

простой service-класс;

обновляем страницу так, чтобы кнопки гарантированно работали.



---

1. Создай файл

/local/modules/its.sitebuilder/lib/bootstrap.php

<?php

namespace Its\Sitebuilder;

use Bitrix\Main\Loader;

class Bootstrap
{
    public static function init(): void
    {
        if (!Loader::includeModule('main')) {
            throw new \RuntimeException('Не удалось подключить модуль main');
        }
    }
}


---

2. Создай файл

/local/modules/its.sitebuilder/lib/service/site/service.php

Сразу создай папки:

/local/modules/its.sitebuilder/lib/service/
/local/modules/its.sitebuilder/lib/service/site/

Файл:

<?php

namespace Its\Sitebuilder\Service\Site;

class Service
{
    public function ping(): array
    {
        global $USER;

        return [
            'ok' => true,
            'module' => 'its.sitebuilder',
            'service' => static::class,
            'userId' => is_object($USER) ? (int)$USER->GetID() : 0,
            'time' => date('c'),
        ];
    }

    public function demoList(): array
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
}


---

3. Создай файл

/local/tools/its.sitebuilder/ajax.php

Сначала создай папку:

/local/tools/its.sitebuilder/

Файл:

<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('NOT_CHECK_PERMISSIONS', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;
use Its\Sitebuilder\Service\Site\Service as SiteService;

header('Content-Type: application/json; charset=UTF-8');

global $USER;

try {
    if (!$USER || !$USER->IsAuthorized()) {
        http_response_code(401);
        echo json_encode([
            'ok' => false,
            'error' => 'NOT_AUTHORIZED',
            'message' => 'Пользователь не авторизован',
        ], JSON_UNESCAPED_UNICODE);
        exit;
    }

    if (!Loader::includeModule('its.sitebuilder')) {
        throw new \RuntimeException('Модуль its.sitebuilder не подключается');
    }

    $action = isset($_REQUEST['action']) ? (string)$_REQUEST['action'] : '';

    $siteService = new SiteService();

    switch ($action) {
        case 'site.ping':
            $result = $siteService->ping();
            break;

        case 'site.demoList':
            $result = $siteService->demoList();
            break;

        default:
            http_response_code(400);
            echo json_encode([
                'ok' => false,
                'error' => 'UNKNOWN_ACTION',
                'message' => 'Неизвестное действие: ' . $action,
            ], JSON_UNESCAPED_UNICODE);
            exit;
    }

    echo json_encode([
        'ok' => true,
        'data' => $result,
    ], JSON_UNESCAPED_UNICODE);
} catch (\Throwable $e) {
    http_response_code(500);

    echo json_encode([
        'ok' => false,
        'error' => 'EXCEPTION',
        'message' => $e->getMessage(),
        'file' => $e->getFile(),
        'line' => $e->getLine(),
    ], JSON_UNESCAPED_UNICODE);
}


---

4. Полностью замени

/local/sitebuilder/index.php

Вот на этот вариант:

<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('NOT_CHECK_PERMISSIONS', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

use Bitrix\Main\Loader;

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

        <p>Промежуточная проверка нового модуля через обычный AJAX endpoint.</p>

        <div style="margin:20px 0;">
            <button id="sb-ping-btn" style="padding:10px 16px;cursor:pointer;">
                Проверить ping
            </button>

            <button id="sb-demo-list-btn" style="padding:10px 16px;cursor:pointer;margin-left:10px;">
                Проверить demoList
            </button>
        </div>

        <pre id="sb-result" style="background:#fff;border:1px solid #d0d7de;padding:16px;white-space:pre-wrap;min-height:220px;">Нажми кнопку для проверки.</pre>
    </div>

    <script>
    (function () {
        var resultNode = document.getElementById('sb-result');
        var pingBtn = document.getElementById('sb-ping-btn');
        var listBtn = document.getElementById('sb-demo-list-btn');

        function printResult(title, data) {
            resultNode.textContent = title + "\n\n" + JSON.stringify(data, null, 2);
        }

        function call(action) {
            printResult('Запрос...', { action: action });

            fetch('/local/tools/its.sitebuilder/ajax.php', {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
                },
                body: 'action=' + encodeURIComponent(action)
            })
            .then(function (response) {
                return response.text().then(function (text) {
                    var parsed = null;

                    try {
                        parsed = JSON.parse(text);
                    } catch (e) {
                        parsed = {
                            ok: false,
                            error: 'INVALID_JSON',
                            raw: text
                        };
                    }

                    printResult('Ответ сервера', {
                        httpStatus: response.status,
                        body: parsed
                    });
                });
            })
            .catch(function (error) {
                printResult('Ошибка fetch', {
                    message: error && error.message ? error.message : String(error)
                });
            });
        }

        pingBtn.addEventListener('click', function () {
            call('site.ping');
        });

        listBtn.addEventListener('click', function () {
            call('site.demoList');
        });

        printResult('Страница загружена', {
            js: 'ok',
            endpoint: '/local/tools/its.sitebuilder/ajax.php'
        });
    })();
    </script>
</body>
</html>


---

Что это нам даст

После этого кнопки обязаны начать что-то показывать, потому что:

мы не зависим от BX.ajax.runAction;

используем обычный fetch;

в <pre> выводится даже сырой ответ сервера;

сразу видно HTTP status и JSON.



---

Что проверить после замены

Открой:

/local/sitebuilder/index.php

И нажми обе кнопки.

Нормальный ответ для ping будет примерно такой:

{
  "httpStatus": 200,
  "body": {
    "ok": true,
    "data": {
      "ok": true,
      "module": "its.sitebuilder",
      "service": "Its\\Sitebuilder\\Service\\Site\\Service",
      "userId": 1,
      "time": "2026-04-09T..."
    }
  }
}


---

Почему я сейчас увёл от D7 controller

Потому что нам сейчас важнее не красота, а поднять рабочий каркас без слепых зон.

Сначала добиваем:

загрузку модуля;

автозагрузку классов;

отдельный ajax-вход;

первый service.


А уже следующим шагом переносим из старого проекта:

site.list

site.create

site.delete


в новый модуль с нормальной структурой.

Когда вставишь это и получишь ответ от кнопок, следующим сообщением я дам уже первый реальный рефакторинг старого api.php: вынесем JsonStorage, SiteRepository и перенесём site.list готовым кодом.