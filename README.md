Обработка заняла 3m 2s

Причина ещё не найдена, но важное уже ясно:

common.bootstrap и common.site в Stage 20 не существуют, поэтому UNKNOWN_ACTION для них ожидаем и не связан с ошибкой редактора.

Редактор реально загружает site.get, затем page.list, а после выбора страницы — block.list.

На скриншоте открыт siteId=14, а успешный page.list ты проверил для siteId=13.


Promise fulfilled: undefined — это нормально, скрипт просто ничего не возвращал.

Проверь реальные запросы редактора

Открой консоль именно на странице:

editor.php?siteId=14

Вставь целиком:

(async function () {
    const config = window.SB_EDITOR_CONFIG || {};

    const currentSiteId = Number(
        config.siteId ||
        new URL(window.location.href).searchParams.get('siteId') ||
        0
    );

    const apiUrl =
        config.apiUrl ||
        '/local/sitebuilder/api/index.php';

    const sessid =
        window.BX && typeof BX.bitrix_sessid === 'function'
            ? BX.bitrix_sessid()
            : String(config.sessid || '');

    async function request(action, data = {}) {
        const body = new URLSearchParams();

        body.set('action', action);
        body.set('sessid', sessid);

        Object.entries(data).forEach(([key, value]) => {
            body.set(key, String(value));
        });

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

        console.group(
            `${action} — HTTP ${response.status}`
        );
        console.log(text);
        console.groupEnd();

        try {
            return JSON.parse(text);
        } catch (error) {
            return null;
        }
    }

    console.clear();
    console.log('Проверяем siteId:', currentSiteId);

    const siteResult = await request(
        'site.get',
        {siteId: currentSiteId}
    );

    if (!siteResult || !siteResult.ok) {
        console.error('ОШИБКА НА ЭТАПЕ site.get');
        return;
    }

    const pagesResult = await request(
        'page.list',
        {siteId: currentSiteId}
    );

    if (!pagesResult || !pagesResult.ok) {
        console.error('ОШИБКА НА ЭТАПЕ page.list');
        return;
    }

    const pages = Array.isArray(pagesResult.pages)
        ? pagesResult.pages
        : [];

    if (!pages.length) {
        console.log(
            'У сайта нет страниц. Основная загрузка API успешна.'
        );
        return;
    }

    const pageId = Number(pages[0].id || 0);

    await request('pageSection.list', {
        siteId: currentSiteId,
        pageId: pageId
    });

    const blocksResult = await request(
        'block.list',
        {pageId: pageId}
    );

    if (!blocksResult || !blocksResult.ok) {
        console.error('ОШИБКА НА ЭТАПЕ block.list');
        return;
    }

    console.log('ВСЕ ОСНОВНЫЕ ЗАПРОСЫ УСПЕШНЫ');
})();

Пришли блок, где будет первая ошибка:

site.get
page.list
или block.list

По этому ответу уже можно будет дать конкретное исправление, а не продолжать проверять миграцию вслепую.