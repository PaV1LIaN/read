Проблема найдена: на странице не подключается JS-ядро Битрикса, поэтому BX вообще не существует.

Значит, \Bitrix\Main\UI\Extension::load('main.core'); в твоём окружении на этой странице не дал нужного результата. Для коробки надёжнее подключить ядро старым способом.

Сделай так.

Замени test_api.php полностью на этот файл:

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';
global $APPLICATION, $USER;

if (method_exists($APPLICATION, 'ShowHead')) {
    // ok
}

CJSCore::Init(['ajax']);

header('Content-Type: text/html; charset=UTF-8');
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder API Test</title>
    <?php $APPLICATION->ShowHead(); ?>
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
        <strong>sessid from PHP:</strong>
        <?= htmlspecialchars(bitrix_sessid(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
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

            function callApi(data) {
                if (typeof window.BX === 'undefined') {
                    print('BX не загружен');
                    return;
                }

                if (typeof BX.ajax !== 'function') {
                    print('BX.ajax не найден');
                    return;
                }

                var sessid = (typeof BX.bitrix_sessid === 'function')
                    ? BX.bitrix_sessid()
                    : '<?= CUtil::JSEscape(bitrix_sessid()) ?>';

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

            document.getElementById('checkBxBtn').addEventListener('click', function () {
                print({
                    BX_exists: typeof window.BX !== 'undefined',
                    BX_ajax_type: typeof window.BX !== 'undefined' ? typeof BX.ajax : 'BX missing',
                    BX_bitrix_sessid_type: typeof window.BX !== 'undefined' ? typeof BX.bitrix_sessid : 'BX missing',
                    sessid_js: (typeof window.BX !== 'undefined' && typeof BX.bitrix_sessid === 'function') ? BX.bitrix_sessid() : null,
                    sessid_php: '<?= CUtil::JSEscape(bitrix_sessid()) ?>'
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
                BX_ajax_type: typeof window.BX !== 'undefined' ? typeof BX.ajax : 'BX missing',
                BX_bitrix_sessid_type: typeof window.BX !== 'undefined' ? typeof BX.bitrix_sessid : 'BX missing',
                sessid_php: '<?= CUtil::JSEscape(bitrix_sessid()) ?>'
            });
        })();
    </script>
</body>
</html>

Если у тебя папка называется не /local/sitebuilder/, а по-другому, замени тут путь:

url: '/local/sitebuilder/api.php',

на свой реальный.

Что изменилось:

добавил CJSCore::Init(['ajax'])

добавил <?php $APPLICATION->ShowHead(); ?> в <head>

оставил fallback на PHP bitrix_sessid(), даже если BX.bitrix_sessid() не будет


После этого снова открой страницу и нажми Проверить BX.

Нормальный результат должен быть примерно такой:

{
  "BX_exists": true,
  "BX_ajax_type": "function",
  "BX_bitrix_sessid_type": "function",
  "sessid_js": "...",
  "sessid_php": "..."
}

Если BX_exists всё ещё будет false, тогда проблема уже не в тестовом файле, а в окружении/шаблоне подключения ядра на этой чистой странице, и тогда я дам тебе второй вариант — вообще без BX.ajax, через обычный fetch, чтобы проверить API без зависимости от JS-ядра Bitrix.