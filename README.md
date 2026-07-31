Обработка заняла 3m 30s

Да. Сейчас заменим два небольших участка, чтобы вместо бесполезного сообщения «Не удалось загрузить редактор» редактор показал точное API-действие и реальный ответ сервера.

Это пока диагностическое исправление: после него станет понятно, какой именно запрос падает.

1. Файл assets/admin/editor/00-core.js

Найди функцию:

function api(action, data) {

Замени её целиком — от function api... до строки перед:

function apiData(res) {

на этот код:

function sbAppendPostValue(params, key, value) {
    if (value === undefined || value === null) {
        return;
    }

    if (Array.isArray(value)) {
        value.forEach(function (item, index) {
            sbAppendPostValue(
                params,
                key + '[' + index + ']',
                item
            );
        });

        return;
    }

    if (typeof value === 'object') {
        Object.keys(value).forEach(function (childKey) {
            sbAppendPostValue(
                params,
                key + '[' + childKey + ']',
                value[childKey]
            );
        });

        return;
    }

    if (typeof value === 'boolean') {
        params.append(key, value ? '1' : '0');
        return;
    }

    params.append(key, String(value));
}

function sbApiErrorMessage(error) {
    if (!error) {
        return 'Неизвестная ошибка';
    }

    var parts = [];

    if (error.action) {
        parts.push('Действие: ' + error.action);
    }

    if (error.error) {
        parts.push('Ошибка: ' + error.error);
    }

    if (error.message) {
        parts.push('Сообщение: ' + error.message);
    }

    if (error.status) {
        parts.push('HTTP: ' + error.status);
    }

    if (error.responseText) {
        parts.push(
            'Ответ сервера: ' +
            String(error.responseText).substring(0, 1500)
        );
    }

    return parts.length
        ? parts.join('\n')
        : JSON.stringify(error, null, 2);
}

function api(action, data) {
    var actionName = String(action || '');

    var isReadOnly =
        /\.(list|get|search|status|health|check)$/i.test(actionName)
        || actionName === 'common.site'
        || actionName === 'common.bootstrap';

    if (typeof setEditorStatus === 'function') {
        setEditorStatus(
            'working',
            isReadOnly ? 'Загрузка…' : 'Сохранение…'
        );
    }

    var params = new URLSearchParams();

    sbAppendPostValue(params, 'action', actionName);
    sbAppendPostValue(params, 'sessid', getSessid());

    Object.keys(data || {}).forEach(function (key) {
        sbAppendPostValue(params, key, data[key]);
    });

    return fetch(API_URL, {
        method: 'POST',
        credentials: 'same-origin',
        headers: {
            'Content-Type':
                'application/x-www-form-urlencoded; charset=UTF-8',
            'X-Requested-With': 'XMLHttpRequest'
        },
        body: params.toString()
    })
        .then(async function (response) {
            var responseText = await response.text();
            var result = null;

            try {
                result = JSON.parse(responseText);
            } catch (parseError) {
                throw {
                    ok: false,
                    error: 'INVALID_JSON_RESPONSE',
                    message:
                        'Сервер вернул не JSON. Возможно, произошла PHP-ошибка.',
                    action: actionName,
                    status: response.status,
                    responseText: responseText
                };
            }

            print(result);

            if (!response.ok || !result || !result.ok) {
                var apiError = Object.assign(
                    {
                        ok: false,
                        action: actionName,
                        status: response.status,
                        responseText: responseText
                    },
                    result || {}
                );

                if (apiError.error === 'VERSION_CONFLICT') {
                    handleVersionConflict(apiError);
                }

                throw apiError;
            }

            if (typeof setEditorStatus === 'function') {
                setEditorStatus(
                    'ready',
                    isReadOnly ? 'Готово' : 'Сохранено'
                );
            }

            return result;
        })
        .catch(function (error) {
            if (typeof setEditorStatus === 'function') {
                setEditorStatus('error', 'Ошибка');
            }

            console.error(
                'SiteBuilder API error:',
                actionName,
                error
            );

            print({
                ok: false,
                action: actionName,
                error: error
            });

            throw error;
        });
}


---

2. Файл assets/admin/editor/60-events.js

В самом конце файла найди:

(async function init() {
    try {
        setManagementPanelsVisible(false);

        await loadSite();
        await loadPages();
        await loadBlocks();
        await loadAccessList();
    } catch (e) {
        print(e);
        alert('Не удалось загрузить редактор');
    }
})();

Замени на:

(async function init() {
    setManagementPanelsVisible(false);

    var steps = [
        {
            name: 'Загрузка сайта',
            action: 'site.get',
            run: loadSite
        },
        {
            name: 'Загрузка страниц',
            action: 'page.list',
            run: loadPages
        },
        {
            name: 'Загрузка секций и блоков',
            action: 'pageSection.list / block.list',
            run: loadBlocks
        },
        {
            name: 'Загрузка прав доступа',
            action: 'access.list',
            run: loadAccessList
        }
    ];

    for (var index = 0; index < steps.length; index++) {
        var step = steps[index];

        try {
            console.log(
                'SiteBuilder init:',
                step.name,
                step.action
            );

            await step.run();
        } catch (error) {
            console.error(
                'SiteBuilder initialization failed:',
                step,
                error
            );

            var errorText =
                typeof sbApiErrorMessage === 'function'
                    ? sbApiErrorMessage(error)
                    : JSON.stringify(error, null, 2);

            print({
                ok: false,
                initializationStep: step.name,
                expectedAction: step.action,
                error: error
            });

            alert(
                'Не удалось загрузить редактор.\n\n' +
                'Этап: ' + step.name + '\n' +
                'Запрос: ' + step.action + '\n\n' +
                errorText
            );

            return;
        }
    }

    console.log('SiteBuilder editor initialized successfully');
})();

3. Обнови версию файлов в editor.php

Найди подключения:

/assets/admin/editor/00-core.js?v=17
/assets/admin/editor/60-events.js?v=17

Замени, например, на:

/assets/admin/editor/00-core.js?v=21
/assets/admin/editor/60-events.js?v=21

Это нужно, чтобы браузер не использовал старый кеш.

4. После замены

Открой редактор и нажми:

Ctrl + F5

Теперь окно должно показать примерно такое:

Этап: Загрузка страниц
Запрос: page.list

Ошибка: INTERNAL_ERROR
HTTP: 500
Ответ сервера: ...

Пришли полный текст нового окна. По нему уже дам конкретное исправление PHP-файла, а не будем гадать.