Обработка заняла 2m 42s

Результат

true
sitebuilder.form_submission

подтверждает, что объекты миграции Stage 20 в базе присутствуют. А:

Promise {fulfilled}
PromiseResult: undefined

не показывает результат API — это лишь значит, что команда завершилась без возвращаемого значения.

Нужно вывести реальный ответ стартовых запросов редактора.

Открой F12 → Console, вставь целиком и нажми Enter:

(async function () {
    const config = window.SB_EDITOR_CONFIG || {};

    const apiUrl =
        config.apiUrl ||
        '/local/sitebuilder/api/index.php';

    const sessid =
        config.sessid ||
        (
            window.BX &&
            typeof BX.bitrix_sessid === 'function'
                ? BX.bitrix_sessid()
                : ''
        );

    const tests = [
        ['common.bootstrap', {
            siteId: config.siteId
        }],
        ['common.site', {
            siteId: config.siteId
        }],
        ['page.list', {
            siteId: config.siteId
        }]
    ];

    for (const [action, data] of tests) {
        const body = new URLSearchParams();

        body.set('action', action);
        body.set('sessid', sessid);

        Object.entries(data).forEach(([key, value]) => {
            body.set(key, String(value ?? ''));
        });

        try {
            const response = await fetch(apiUrl, {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type':
                        'application/x-www-form-urlencoded; charset=UTF-8'
                },
                body: body.toString()
            });

            const text = await response.text();

            console.log(
                '\n===== ' + action + ' =====\n' +
                'HTTP: ' + response.status + '\n' +
                text
            );
        } catch (error) {
            console.error(action, error);
        }
    }
})();

В консоли появятся три блока:

===== common.bootstrap =====
===== common.site =====
===== page.list =====

Нужен блок, где:

HTTP не 200;

либо в JSON есть "ok": false;

либо появляется PHP warning, HTML или пустой ответ.


Скопируй сюда все три результата. После выполнения снова может появиться Promise {pending} или fulfilled undefined — это нормально, важны строки, которые скрипт выведет выше.