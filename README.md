По последнему архиву вижу: текущий editor.php уже 2647 строк, и в нём всё ещё один большой BX.ready(...).
Рефакторить его правильно можно, но не “разрезая вслепую”. У тебя почти все функции завязаны на локальные переменные внутри BX.ready:

siteId

pageId

blocksBox

blockSearch

collapsedBlocks

кнопки btnAdd...


Из-за этого сразу вынести готовые .js файлы без промежуточного шага рискованно.

Что я предлагаю как правильный следующий шаг

Сначала сделать общий объект состояния редактора, и уже потом раскладывать функции по файлам.


---

Какой должна быть новая структура JS

/local/sitebuilder/assets/js/
  editor.core.js
  editor.api.js
  editor.blocks.js
  editor.sections.js
  editor.dnd.js
  editor.dialogs.js
  editor.init.js


---

Что реально есть сейчас в твоём editor.php

Я посмотрел текущий файл, и там уже выделяются такие блоки логики:

API / базовые helper-функции

notify

api

fileDownloadUrl

getFilesForSite

btnClass

headingTag

headingAlign

colsGridTemplate

galleryTemplate

cardsNormalizeItem


Рендер

buildBlockShell

renderBlocks


DnD / порядок

saveBlockOrder

initBlockDnD


Section

SECTION_PRESETS

sectionPresetOptions

applySectionPresetToForm

createBlockAfterSection

quickAddHeadingAfterSection

quickAddTextAfterSection

quickAddButtonAfterSection

quickAddCardsAfterSection

addSectionBlock

editSectionBlock


Диалоги обычных блоков

addTextBlock

addImageBlock

addButtonBlock

addHeadingBlock

addCols2Block

addSpacerBlock

addGalleryBlock

addCardBlock

addCardsBlock

editTextBlock

editImageBlock

editButtonBlock

editHeadingBlock

editCols2Block

editSpacerBlock

editGalleryBlock

editCardBlock

editCardsBlock


Инициализация / события

loadBlocks

saveTemplateFromPage

applyTemplateToPage

openSectionsLibrary

большой document.addEventListener('click', ...)

привязки btnAdd...addEventListener(...)

loadBlocks();



---

Правильный первый шаг сейчас

Сделать editor.core.js

Он должен создать глобальный namespace и общее состояние редактора.

Готовое содержимое assets/js/editor.core.js

window.SBEditor = window.SBEditor || {};

window.SBEditor.state = {
  siteId: 0,
  pageId: 0,
  blocksBox: null,
  blockSearch: null,
  collapsedBlocks: new Set(),

  btnAddSection: null,
  btnAddText: null,
  btnAddImage: null,
  btnAddButton: null,
  btnAddHeading: null,
  btnAddCols2: null,
  btnAddGallery: null,
  btnAddSpacer: null,
  btnAddCard: null,
  btnAddCards: null,

  btnSaveTemplate: null,
  btnApplyTemplate: null,
  btnSections: null,
};

window.SBEditor.setState = function (patch) {
  Object.assign(window.SBEditor.state, patch || {});
};

window.SBEditor.getState = function () {
  return window.SBEditor.state;
};


---

Что делать потом

После этого уже можно по-настоящему выносить функции.

Порядок выноса, который я рекомендую

1. editor.core.js


2. editor.api.js


3. editor.sections.js


4. editor.dnd.js


5. editor.blocks.js


6. editor.dialogs.js


7. editor.init.js




---

Почему не стоит сразу выносить всё

Потому что сейчас если просто копировать функции по файлам, они начнут падать на:

blocksBox is not defined

pageId is not defined

notify is not defined

collapsedBlocks is not defined


То есть сначала нужен единый shared state, а уже потом — нормальный разнос по файлам.


---

Что бы я делал прямо сейчас

Создай файл:

/local/sitebuilder/assets/js/editor.core.js

и вставь туда код выше.

После этого следующим сообщением я дам тебе готовый editor.api.js и покажу, как уже начать подключать эти файлы в editor.php без поломки.