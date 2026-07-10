В 60-events.js исправлять ничего не нужно. Обработчик выбора страницы написан правильно:

state.currentPageId = Number(item.getAttribute('data-page-id') || 0);

После этого он перерисовывает страницы и загружает блоки:

renderPages();
fillPageForm();
await loadBlocks();

В присланном файле нет строки, которая после клика намеренно возвращает currentPageId к 14. Значит нужно точно найти, какой код выполняет повторное присваивание.

Вставь в консоль этот перехватчик:

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

После сообщения:

Перехват установлен

обычным кликом нажми страницу «Тест» в левом меню.

Через две секунды выполни:

setTimeout(function () {
    console.log({
        currentPageId: state.currentPageId,
        pageTitle: document.getElementById('canvasPageTitle')?.textContent,
        activePageId: document.querySelector(
            '#pagesList .sb-editor-page-item.is-active'
        )?.dataset.pageId,
        blockIds: state.blocks.map(function (block) {
            return Number(block.id || 0);
        })
    });
}, 2000);

В консоли появятся строки примерно такого вида:

[currentPageId] 14 → 31

Если происходит настоящий возврат, появится ещё одна строка:

[currentPageId] 31 → 14

Под ней будет стек с точным именем файла и номером строки, которая возвращает страницу на 14.

Пока не меняй ни 20-pages.js, ни 60-events.js. По представленному коду причина возврата в них не подтверждается. После диагностики страницу достаточно перезагрузить, чтобы удалить временный перехватчик.