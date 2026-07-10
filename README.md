console.log({
    configApiUrl: window.SB_EDITOR_CONFIG.apiUrl,
    actualApiUrl: API_URL,
    siteId: siteId
});
undefined
loadPages().then(function () {
    console.log('Страницы:', state.pages);
});
Promise {<pending>}
[[Prototype]]
: 
Promise
[[PromiseState]]
: 
"fulfilled"
[[PromiseResult]]
: 
undefined
state.currentPageId = 31;

loadBlocks().then(function () {
    console.log('Блоки:', state.blocks);
});
Promise {<pending>}
[[Prototype]]
: 
Promise
[[PromiseState]]
: 
"fulfilled"
[[PromiseResult]]
: 
undefined
