Скорее всего проблема в одном из трёх мест:

1. не загрузился BX


2. JS падает с ошибкой и дальше не выполняется


3. файл открывается, но BX.ajax не существует



Сделай так: замени тестовый файл полностью на этот вариант. Я убрал лишнее и добавил явную диагностику.

/local/sitebuilder/test_api.php

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
\Bitrix\Main\UI\Extension::load('main.core');
global $USER;
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder API Test</title>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; }
        button { margin: 0 10px 10px 0; padding: 10px 14px; cursor: pointer; }
        pre { white-space: pre-wrap; background: #f5f5f5; padding: 16px; border: 1px solid #ccc; min-height: 180px; }
        .row { margin-bottom: 12px; }
    </style>
</head>
<body>
    <h1>Проверка API SiteBuilder</h1>

    <div class="row">
        <strong>Пользователь:</strong>
        <?= htmlspecialchars((string)$USER->GetLogin(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        (ID <?= (int)$USER->GetID() ?>)
    </div>

    <div class="row">
        <strong>sessid:</strong>
        <span id="sessidView"></span>
    </div>

    <div class="row">
        <button type="button" id="checkBxBtn">Проверить BX</button>
        <button type="button" id="pingBtn">Ping</button>
        <button type="button" id="siteListBtn">Site list</button>
    </div>

    <pre id="out">Здесь будет результат...</pre>

    <script>
        (function () {
            var out = document.getElementById('out');

            function print(data) {
                if (typeof data === 'string') {
                    out.textContent = data;
                    return;
                }
                try {
                    out.textContent = JSON.stringify(data, null, 2);
                } catch (e) {
                    out.textContent = String(data);
                }
            }

            function printError(prefix, err) {
                var text = prefix + '\n';
                try {
                    text += JSON.stringify(err, null, 2);
                } catch (e) {
                    text += String(err);
                }
                out.textContent = text;
            }

            function hasBx() {
                return typeof window.BX !== 'undefined';
            }

            function callApi(data) {
                if (!hasBx()) {
                    print('BX не загружен');
                    return;
                }

                if (typeof BX.ajax !== 'function') {
                    print('BX.ajax не найден');
                    return;
                }

                var sessid = BX.bitrix_sessid ? BX.bitrix_sessid() : '';
                if (!sessid) {
                    print('Не удалось получить sessid');
                    return;
                }

                print({
                    status: 'sending',
                    url: '/local/sitebuilder/api.php',
                    data: Object.assign({ sessid: sessid }, data)
                });

                BX.ajax({
                    url: '/local/sitebuilder/api.php',
                    method: 'POST',
                    data: Object.assign({
                        sessid: sessid
                    }, data),
                    dataType: 'json',
                    timeout: 30,
                    onsuccess: function (res) {
                        print(res);
                    },
                    onfailure: function (err) {
                        printError('AJAX ERROR', err);
                    }
                });
            }

            window.onerror = function (message, source, lineno, colno, error) {
                print({
                    jsError: true,
                    message: message,
                    source: source,
                    line: lineno,
                    column: colno,
                    stack: error && error.stack ? error.stack : null
                });
            };

            document.getElementById('sessidView').textContent =
                (hasBx() && BX.bitrix_sessid) ? BX.bitrix_sessid() : 'BX или BX.bitrix_sessid не загружен';

            document.getElementById('checkBxBtn').addEventListener('click', function () {
                print({
                    BX_exists: typeof window.BX !== 'undefined',
                    BX_ajax_type: hasBx() ? typeof BX.ajax : 'BX missing',
                    BX_bitrix_sessid_type: hasBx() ? typeof BX.bitrix_sessid : 'BX missing',
                    sessid: (hasBx() && BX.bitrix_sessid) ? BX.bitrix_sessid() : null
                });
            });

            document.getElementById('pingBtn').addEventListener('click', function () {
                callApi({ action: 'ping' });
            });

            document.getElementById('siteListBtn').addEventListener('click', function () {
                callApi({ action: 'site.list' });
            });

            print({
                loaded: true,
                BX_exists: typeof window.BX !== 'undefined',
                BX_ajax_type: hasBx() ? typeof BX.ajax : 'BX missing',
                BX_bitrix_sessid_type: hasBx() ? typeof BX.bitrix_sessid : 'BX missing'
            });
        })();
    </script>
</body>
</html>

Что проверить сразу

Открой:

/local/sitebuilder/test_api.php

И посмотри, что написано в сером блоке сразу после загрузки.

Нормальный вариант

Там должно быть что-то вроде:

{
  "loaded": true,
  "BX_exists": true,
  "BX_ajax_type": "function",
  "BX_bitrix_sessid_type": "function"
}

Если там BX missing

Значит не подгрузился Bitrix JS.

Тогда замени вверху файла:

\Bitrix\Main\UI\Extension::load('main.core');

на более совместимый вариант:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
CJSCore::Init(['ajax']);
global $USER;
?>

То есть весь верх файла должен стать таким:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
CJSCore::Init(['ajax']);
global $USER;
?>

Если кнопки всё равно “молчат”

Открой консоль браузера:

F12

вкладка Console


И пришли мне:

что написано в сером блоке на странице

есть ли ошибка в Console

что происходит при нажатии на “Проверить BX”


Ещё одна частая причина

Если твоя новая чистая папка называется не /local/sitebuilder/, а по-другому, то вот тут надо поменять путь:

url: '/local/sitebuilder/api.php',

Например, если папка называется /local/sitebuilder_new/, должно быть:

url: '/local/sitebuilder_new/api.php',

Это нужно заменить в двух местах внутри файла.

Быстрый чек без консоли

Нажми кнопку Проверить BX.

Если всё хорошо, увидишь примерно это:

{
  "BX_exists": true,
  "BX_ajax_type": "function",
  "BX_bitrix_sessid_type": "function",
  "sessid": "..."
}

Если увидишь:

BX missing — проблема в подключении Bitrix JS

BX.ajax not found — проблема в ядре JS

sending, но дальше ошибка — проблема уже в api.php


Пришли мне либо текст из серого блока, либо скрин консоли, и я скажу точечно, что именно чинить.