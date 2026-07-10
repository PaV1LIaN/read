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
undefined
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
{currentPageId: 31, pageTitle: 'Тест', activePageId: '31', blockIds: Array(2)}
activePageId
: 
"31"
blockIds
: 
(2) [80, 81]
currentPageId
: 
31
pageTitle
: 
"Тест"
[[Prototype]]
: 
Object
