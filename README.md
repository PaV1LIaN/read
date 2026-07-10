Это нормальное поведение консоли, ошибок здесь нет.

console.log(...) всегда возвращает undefined. Сам объект выводится отдельной строкой выше.

loadPages() и loadBlocks() тоже возвращают undefined, потому что в конце этих функций нет оператора return. Состояние они записывают в state.

Выполни команды без console.log, чтобы консоль показала сами значения.

1. Проверка адреса API

({
    configApiUrl: window.SB_EDITOR_CONFIG.apiUrl,
    actualApiUrl: API_URL,
    siteId: siteId
})

Ожидаемый результат:

{
    configApiUrl: "/local/sitebuilder/api/index.php",
    actualApiUrl: "/local/sitebuilder/api/index.php",
    siteId: 13
}

2. Проверка загруженных страниц

state.pages

Должен отобразиться массив из двух страниц.

Для компактного вывода:

state.pages.map(function (page) {
    return {
        id: page.id,
        title: page.title,
        parentId: page.parentId,
        status: page.status
    };
})

3. Проверка текущей страницы

state.currentPageId

После твоей команды должно быть:

31

4. Проверка блоков

state.blocks

Должен быть массив из двух блоков:

ID 80 — heading
ID 81 — text

Компактная проверка:

state.blocks.map(function (block) {
    return {
        id: block.id,
        pageId: block.pageId,
        type: block.type,
        sort: block.sort
    };
})

5. Проверка фактической отрисовки

document.getElementById('pagesList').innerHTML

и:

document.getElementById('blocksList').innerHTML

Если в этих двух командах есть HTML-код страниц и блоков, значит данные загружены и отрисованы.

Также проверь количество созданных элементов:

({
    pageElements: document.querySelectorAll('#pagesList .sb-editor-page-item').length,
    blockElements: document.querySelectorAll('#blocksList [data-block-id]').length,
    pagesInState: state.pages.length,
    blocksInState: state.blocks.length
})

Для страниц ожидается:

pageElements: 2
pagesInState: 2

Для блоков blocksInState должно быть 2. Значение blockElements зависит от того, какой атрибут создаёт функция renderBlocks().

Сейчас по присланному результату подтверждено:

loadPages() завершилась успешно;

loadBlocks() завершилась успешно;

JavaScript-исключений внутри этих функций не произошло;

undefined не является ошибкой.


Следующий нужный результат — вывод этих трёх команд:

({
    apiUrl: API_URL,
    currentPageId: state.currentPageId,
    pagesCount: state.pages.length,
    blocksCount: state.blocks.length
})

document.getElementById('pagesList').innerHTML

document.getElementById('blocksList').innerHTML