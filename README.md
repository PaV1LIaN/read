Скорее всего после замены сломалось одно из двух:

1. page.create не доходит до handler-а


2. в editor.php JS падает раньше, чем вешается обработчик на кнопку



Чтобы не гадать, давай починим быстро и по месту.


---

Что проверить первым делом

Открой editor.php, нажми Создать и посмотри:

есть ли что-то в блоке Отладка

есть ли ошибка в Console

есть ли запрос в Network на api.php


Но чтобы не тратить время, ниже даю более надёжную версию кода для editor.php, где:

кнопка точно вешается

ошибки по create лучше видны

создание страницы работает даже если часть логики выше упала



---

Что заменить в editor.php

Найди внизу файла блок, где сейчас стоят обработчики:

document.getElementById('createPageBtn').addEventListener('click', createPage);
document.getElementById('savePageParentBtn').addEventListener('click', saveCurrentPageParent);
document.getElementById('reloadBlocksBtn').addEventListener('click', loadBlocks);
document.getElementById('saveBlockBtn').addEventListener('click', saveBlock);
document.getElementById('deleteBlockBtn').addEventListener('click', function () {
    if (state.currentBlockId) {
        deleteBlock(state.currentBlockId);
    }
});
document.getElementById('formatJsonBtn').addEventListener('click', formatCurrentJson);
document.getElementById('resetPropsBtn').addEventListener('click', resetProps);

И замени целиком на это:

var createPageBtn = document.getElementById('createPageBtn');
var savePageParentBtn = document.getElementById('savePageParentBtn');
var reloadBlocksBtn = document.getElementById('reloadBlocksBtn');
var saveBlockBtn = document.getElementById('saveBlockBtn');
var deleteBlockBtn = document.getElementById('deleteBlockBtn');
var formatJsonBtn = document.getElementById('formatJsonBtn');
var resetPropsBtn = document.getElementById('resetPropsBtn');

if (createPageBtn) {
    createPageBtn.addEventListener('click', function (e) {
        e.preventDefault();
        try {
            createPage();
        } catch (err) {
            print({
                ok: false,
                error: 'CREATE_PAGE_JS_ERROR',
                message: err && err.message ? err.message : String(err),
                stack: err && err.stack ? err.stack : null
            });
            alert('Ошибка JS при создании страницы');
        }
    });
}

if (savePageParentBtn) {
    savePageParentBtn.addEventListener('click', function (e) {
        e.preventDefault();
        try {
            saveCurrentPageParent();
        } catch (err) {
            print({
                ok: false,
                error: 'SAVE_PARENT_JS_ERROR',
                message: err && err.message ? err.message : String(err),
                stack: err && err.stack ? err.stack : null
            });
        }
    });
}

if (reloadBlocksBtn) {
    reloadBlocksBtn.addEventListener('click', function (e) {
        e.preventDefault();
        loadBlocks();
    });
}

if (saveBlockBtn) {
    saveBlockBtn.addEventListener('click', function (e) {
        e.preventDefault();
        saveBlock();
    });
}

if (deleteBlockBtn) {
    deleteBlockBtn.addEventListener('click', function (e) {
        e.preventDefault();
        if (state.currentBlockId) {
            deleteBlock(state.currentBlockId);
        }
    });
}

if (formatJsonBtn) {
    formatJsonBtn.addEventListener('click', function (e) {
        e.preventDefault();
        formatCurrentJson();
    });
}

if (resetPropsBtn) {
    resetPropsBtn.addEventListener('click', function (e) {
        e.preventDefault();
        resetProps();
    });
}


---

Ещё нужно заменить саму функцию createPage()

Найди функцию:

function createPage() {
    ...
}

И замени её целиком на эту:

function createPage() {
    var titleInput = document.getElementById('newPageTitle');
    var slugInput = document.getElementById('newPageSlug');
    var parentSelect = document.getElementById('newPageParentId');

    if (!titleInput) {
        print({
            ok: false,
            error: 'TITLE_INPUT_NOT_FOUND'
        });
        alert('Не найдено поле названия страницы');
        return;
    }

    var title = (titleInput.value || '').trim();
    var slug = slugInput ? (slugInput.value || '').trim() : '';
    var parentId = parentSelect ? (parseInt(parentSelect.value, 10) || 0) : 0;

    if (!title) {
        alert('Введите название страницы');
        titleInput.focus();
        return;
    }

    print({
        debug: true,
        step: 'page.create.request',
        siteId: SITE_ID,
        title: title,
        slug: slug,
        parentId: parentId
    });

    api('page.create', {
        siteId: SITE_ID,
        title: title,
        slug: slug,
        parentId: parentId
    }, function (res) {
        if (!res || res.ok !== true) {
            alert('Не удалось создать страницу');
            return;
        }

        titleInput.value = '';
        if (slugInput) {
            slugInput.value = '';
        }
        if (parentSelect) {
            parentSelect.value = '0';
        }

        state.currentPageId = Number((res.page && res.page.id) || 0);
        loadPages(function () {
            loadBlocks();
        });
    }, function (err) {
        print({
            ok: false,
            error: 'PAGE_CREATE_FAILURE',
            detail: err
        });
        alert('Ошибка запроса при создании страницы');
    });
}


---

Если после этого всё ещё не создаёт

Тогда проблема почти точно в роутинге api/index.php.

Проверь, есть ли в файле:

/local/sitebuilder/api/index.php

вот такой блок:

if (strpos($action, 'page.') === 0) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

Если его нет — добавь обязательно.

Лучше поставить рядом с другими handler-ами, например так:

if (strpos($action, 'site.') === 0) {
    require __DIR__ . '/handlers/site.php';
    exit;
}

if (strpos($action, 'page.') === 0) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

if (strpos($action, 'block.') === 0) {
    require __DIR__ . '/handlers/block.php';
    exit;
}


---

Что должно появиться после нажатия

После нажатия Создать в блоке Отладка должно быть что-то вроде:

{
  "debug": true,
  "step": "page.create.request",
  "siteId": 1,
  "title": "Новая страница",
  "slug": "",
  "parentId": 0
}

А затем ответ API.


---

Самая вероятная причина у тебя сейчас

Если кнопка вообще “как будто ничего не делает”, то чаще всего это:

JS упал раньше на одном из getElementById(...).addEventListener(...)

или page.* не маршрутизируется в api/index.php


Сначала замени два JS-фрагмента выше.
Если после этого в отладке появится ответ UNKNOWN_ACTION или NOT_MOVED_YET, значит следующим сообщением я дам тебе точный готовый api/index.php под твою структуру.