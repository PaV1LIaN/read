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
{currentPageId: 31, blocks: Array(2)}
blocks
: 
(2) [{…}, {…}]
currentPageId
: 
31
[[Prototype]]
: 
Object
