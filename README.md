Обработка заняла 2m 44s

Миграцию повторно не запускай. Сообщение слишком общее: редактор скрывает настоящую ошибку внутри стартовой цепочки site.get → page.list → block.list. Нужно увидеть, какой запрос падает.

1. Быстро проверь миграцию в pgAdmin

Выполни:

SELECT
    EXISTS (
        SELECT 1
        FROM information_schema.columns
        WHERE table_schema = 'sitebuilder'
          AND table_name = 'page'
          AND column_name = 'seo_json'
    ) AS seo_json_exists,
    to_regclass('sitebuilder.form_submission') AS form_submission_table;

Нормальный результат:

seo_json_exists = true
form_submission_table = sitebuilder.form_submission

2. Найди падающий API-запрос через браузер

На странице редактора нажми F12 → вкладка Console / Консоль.

Вставь целиком этот код:

(async () => {
    const siteId = 14;

    async function check(action, data = {}) {
        const body = new URLSearchParams({
            action,
            sessid: BX.bitrix_sessid(),
            ...Object.fromEntries(
                Object.entries(data).map(([key, value]) => [key, String(value)])
            )
        });

        const response = await fetch(
            '/local/sitebuilder/api/index.php',
            {
                method: 'POST',
                credentials: 'same-origin',
                headers: {
                    'Content-Type':
                        'application/x-www-form-urlencoded; charset=UTF-8'
                },
                body
            }
        );

        const text = await response.text();

        let result;

        try {
            result = JSON.parse(text);
        } catch (error) {
            result = text;
        }

        console.log(
            '%c' + action,
            'font-weight:bold',
            'HTTP ' + response.status,
            result
        );

        return {
            ok: response.ok && result && result.ok,
            result
        };
    }

    let checkResult = await check('site.get', { siteId });

    if (!checkResult.ok) {
        return;
    }

    checkResult = await check('page.list', { siteId });

    if (!checkResult.ok) {
        return;
    }

    const pages = Array.isArray(checkResult.result.pages)
        ? checkResult.result.pages
        : [];

    const pageId = Number(pages[0]?.id || 0);

    if (!pageId) {
        console.log('У сайта нет страниц');
        return;
    }

    await check('pageSection.list', {
        siteId,
        pageId
    });

    await check('block.list', {
        pageId
    });

    await check('site.accessList', {
        siteId
    });
})();

Если Chrome запрещает вставку, вручную набери в консоли:

allow pasting

и повтори вставку.

В консоли появятся строки примерно такие:

site.get HTTP 200 {ok: true, ...}
page.list HTTP 500 {ok: false, error: "INTERNAL_ERROR"}

Пришли:

1. результат запроса из pgAdmin;


2. первую строку консоли, где ok: false или HTTP не 200;


3. раскрытый JSON этой ошибки.



После этого будет понятно, исправлять ли БД, права или конкретный файл API.