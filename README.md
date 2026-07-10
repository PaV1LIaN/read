На странице:

/local/sitebuilder/editor.php?siteId=13

открой консоль браузера через F12 → вкладка Console.

Вставь этот код и нажми Enter:

(function () {
    var currentValue = Number(state.currentPageId || 0);

    Object.defineProperty(state, 'currentPageId', {
        configurable: true,
        enumerable: true,

        get: function () {
            return currentValue;
        },

        set: function (newValue) {
            console.log(
                '[currentPageId]',
                currentValue,
                '→',
                newValue,
                '\n',
                new Error().stack
            );

            currentValue = Number(newValue || 0);
        }
    });

    console.log('Перехват установлен. Текущее значение:', currentValue);
})();

После этого не вводи другие команды сразу.

Обычным кликом в левом списке страниц нажми страницу «Тест».

Затем вставь в консоль:

({
    currentPageId: state.currentPageId,
    pageTitle: document.getElementById('canvasPageTitle')?.textContent,
    activePageId: document.querySelector(
        '#pagesList .sb-editor-page-item.is-active'
    )?.dataset.pageId,
    blockIds: state.blocks.map(function (block) {
        return Number(block.id || 0);
    })
})

Правильный результат:

{
    currentPageId: 31,
    pageTitle: "Тест",
    activePageId: "31",
    blockIds: [80, 81]
}

Также посмотри, появились ли в консоли строки:

[currentPageId] 14 → 31

или:

[currentPageId] 31 → 14

Нужен результат второй команды и весь лог [currentPageId], который появится после клика по странице «Тест».