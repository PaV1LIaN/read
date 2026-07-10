По результатам проверки сейчас всё загружается и отрисовывается корректно:

API используется правильный: /local/sitebuilder/api/index.php;

страницы загружены: 2;

блоки текущей страницы загружены: 2;

HTML страниц создан;

HTML блоков создан;

ошибок JavaScript нет.


20-pages.js и 00-core.js менять не нужно.

Текущая выбранная страница:

state.currentPageId
// 14

Поэтому в state.blocks находятся блоки страницы «Диск»:

78 — disk
85 — text

Это правильное поведение. В loadPages() первая страница выбирается автоматически:

if (!state.currentPageId && state.pages.length) {
    state.currentPageId = Number(state.pages[0].id || 0);
}

Следующая проверка

Нужно проверить переключение страницы кликом. Выполни:

document.querySelector('[data-page-id="31"]').click();

Затем:

({
    currentPageId: state.currentPageId,
    blocks: state.blocks.map(function (block) {
        return {
            id: block.id,
            pageId: block.pageId,
            type: block.type
        };
    })
})

Правильный результат должен быть примерно таким:

{
    currentPageId: 31,
    blocks: [
        {
            id: 80,
            pageId: 31,
            type: "heading"
        },
        {
            id: 81,
            pageId: 31,
            type: "text"
        }
    ]
}

Если после клика currentPageId останется 14, проблема находится в обработчике клика в:

/local/sitebuilder/assets/admin/editor/60-events.js

Если станет 31 и загрузятся блоки 80 и 81, то редактор страниц работает полностью, и можно переходить к следующему функционалу проекта.