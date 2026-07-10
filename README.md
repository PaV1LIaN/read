
({
    configApiUrl: window.SB_EDITOR_CONFIG.apiUrl,
    actualApiUrl: API_URL,
    siteId: siteId
})
{configApiUrl: '/local/sitebuilder/api/index.php', actualApiUrl: '/local/sitebuilder/api/index.php', siteId: 13}
actualApiUrl
: 
"/local/sitebuilder/api/index.php"
configApiUrl
: 
"/local/sitebuilder/api/index.php"
siteId
: 
13
[[Prototype]]
: 
Object
state.pages.map(function (page) {
    return {
        id: page.id,
        title: page.title,
        parentId: page.parentId,
        status: page.status
    };
})
(2) [{…}, {…}]
0
: 
{id: 14, title: 'Диск', parentId: 0, status: 'published'}
1
: 
{id: 31, title: 'Тест', parentId: 0, status: 'published'}
length
: 
2
[[Prototype]]
: 
Array(0)
state.currentPageId
14
state.blocks
(2) [{…}, {…}]
0
: 
{id: 78, pageId: 14, type: 'disk', sort: 40, content: Array(0), …}
1
: 
{id: 85, pageId: 14, type: 'text', sort: 50, content: {…}, …}
length
: 
2
[[Prototype]]
: 
Array(0)
state.blocks.map(function (block) {
    return {
        id: block.id,
        pageId: block.pageId,
        type: block.type,
        sort: block.sort
    };
})
(2) [{…}, {…}]
0
: 
{id: 78, pageId: 14, type: 'disk', sort: 40}
1
: 
{id: 85, pageId: 14, type: 'text', sort: 50}
length
: 
2
[[Prototype]]
: 
Array(0)
document.getElementById('pagesList').innerHTML
'<div class="sb-editor-page-item is-active" data-page-id="14" style="margin-left:0px;">  <div class="sb-editor-page-top">      <div>          <h3 class="sb-editor-page-title">Диск</h3>          <div class="sb-editor-page-meta"><span class="sb-editor-chip">disk</span><span class="sb-editor-chip sb-editor-chip--green">published</span>          </div>      </div>  </div></div><div class="sb-editor-page-item" data-page-id="31" style="margin-left:0px;">  <div class="sb-editor-page-top">      <div>          <h3 class="sb-editor-page-title">Тест</h3>          <div class="sb-editor-page-meta"><span class="sb-editor-chip">test</span><span class="sb-editor-chip sb-editor-chip--green">published</span>          </div>      </div>  </div></div>'
document.getElementById('blocksList').innerHTML
'<div class="sb-editor-section-preview is-active" data-editor-section-id="4">  <div class="sb-editor-section-preview__head" data-page-section-select="4">      <div>          <h3 class="sb-editor-section-preview__title">Основная секция</h3>          <div class="sb-editor-section-preview__meta">              <span>1 кол.</span>              <span>default</span>          </div>      </div>      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-add-block-to-section="4">Выбрать</button>  </div>  <div class="sb-editor-section-preview__grid sb-editor-section-preview__grid--1"><div class="sb-editor-section-preview__column is-target" data-section-id="4" data-column="1">  <div class="sb-editor-section-preview__column-head">      <div class="sb-editor-section-preview__column-title">Колонка 1</div>      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-set-add-target="4" data-column="1">Выбрано      </button>  </div><div class="sb-editor-block" draggable="true" data-block-id="78">  <div class="sb-editor-block-head">      <div>          <h3 class="sb-editor-block-title">disk</h3>          <div class="sb-editor-chip">block #78</div>      </div>  </div>  <div class="sb-editor-block-preview">Компонент "Диск": Файлы · rootMode=block · view=table · секция #4 · колонка 1</div></div><div class="sb-editor-block" draggable="true" data-block-id="85">  <div class="sb-editor-block-head">      <div>          <h3 class="sb-editor-block-title">text</h3>          <div class="sb-editor-chip">block #85</div>      </div>  </div>  <div class="sb-editor-block-preview">dfsdsddfdfdf · секция #4 · колонка 1</div></div></div>  </div></div>'
({
    pageElements: document.querySelectorAll('#pagesList .sb-editor-page-item').length,
    blockElements: document.querySelectorAll('#blocksList [data-block-id]').length,
    pagesInState: state.pages.length,
    blocksInState: state.blocks.length
})
{pageElements: 2, blockElements: 2, pagesInState: 2, blocksInState: 2}
blockElements
: 
2
blocksInState
: 
2
pageElements
: 
2
pagesInState
: 
2
[[Prototype]]
: 
Object
constructor
: 
ƒ Object()
hasOwnProperty
: 
ƒ hasOwnProperty()
isPrototypeOf
: 
ƒ isPrototypeOf()
propertyIsEnumerable
: 
ƒ propertyIsEnumerable()
toLocaleString
: 
ƒ toLocaleString()
toString
: 
ƒ toString()
valueOf
: 
ƒ valueOf()
__defineGetter__
: 
ƒ __defineGetter__()
__defineSetter__
: 
ƒ __defineSetter__()
__lookupGetter__
: 
ƒ __lookupGetter__()
__lookupSetter__
: 
ƒ __lookupSetter__()
__proto__
: 
(...)
get __proto__
: 
ƒ __proto__()
set __proto__
: 
ƒ __proto__()
