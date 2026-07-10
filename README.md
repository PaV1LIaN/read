({
    pageTitle: document.getElementById('canvasPageTitle')?.textContent,
    activePageId: document.querySelector('#pagesList .sb-editor-page-item.is-active')?.dataset.pageId,
    renderedBlocks: Array.from(
        document.querySelectorAll('#blocksList [data-block-id]')
    ).map(function (element) {
        return Number(element.dataset.blockId);
    })
})
{pageTitle: 'Диск', activePageId: '14', renderedBlocks: Array(2)}
activePageId
: 
"14"
pageTitle
: 
"Диск"
renderedBlocks
: 
(2) [78, 85]
[[Prototype]]
: 
Object
