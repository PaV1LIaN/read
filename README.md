Переключение страницы работает корректно:

currentPageId изменился с 14 на 31;

loadBlocks() загрузил два блока страницы 31;

обработчик клика в 60-events.js исправен;

20-pages.js и 00-core.js больше менять не нужно.


Теперь проверь, что после переключения обновился именно интерфейс, а не только state:

({
    pageTitle: document.getElementById('canvasPageTitle')?.textContent,
    activePageId: document.querySelector('#pagesList .sb-editor-page-item.is-active')?.dataset.pageId,
    renderedBlocks: Array.from(
        document.querySelectorAll('#blocksList [data-block-id]')
    ).map(function (element) {
        return Number(element.dataset.blockId);
    })
})

Для страницы «Тест» должен быть результат:

{
    pageTitle: "Тест",
    activePageId: "31",
    renderedBlocks: [80, 81]
}

Если результат такой, весь цикл редактора подтверждён:

page.list
→ отображение дерева страниц
→ выбор страницы
→ block.list
→ обновление state
→ отрисовка блоков

Исправление apiUrl в editor.php было нужным. Остальной проверенный код оставляем без изменений.