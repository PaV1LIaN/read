Сейчас самый безопасный следующий шаг — сделать переходный мост, чтобы потом вынести dialogs почти без переписывания.

Почему: в текущем editor.php блок диалогов жёстко завязан на локальные переменные внутри BX.ready(...):

siteId

pageId

api

notify

loadBlocks

blocksBox

collapsedBlocks


Если сразу резать диалоги, легко всё сломать. Поэтому сначала делаем bridge.

1. Создай файл

/local/sitebuilder/assets/js/editor.bridge.js

и вставь:

window.SBEditor = window.SBEditor || {};

window.SBEditor.syncLegacyBridge = function () {
  const st = window.SBEditor.getState();

  window.siteId = st.siteId;
  window.pageId = st.pageId;
  window.blocksBox = st.blocksBox;
  window.blockSearch = st.blockSearch;
  window.collapsedBlocks = st.collapsedBlocks;
  window.allPageBlocks = st.allPageBlocks || [];

  window.notify = window.SBEditor.notify;
  window.api = window.SBEditor.api;

  window.fileDownloadUrl = function (fileId) {
    return window.SBEditor.fileDownloadUrl(window.SBEditor.getState().siteId, fileId);
  };

  window.getFilesForSite = function () {
    return window.SBEditor.getFilesForSite();
  };

  window.btnClass = window.SBEditor.btnClass;
  window.headingTag = window.SBEditor.headingTag;
  window.headingAlign = window.SBEditor.headingAlign;
  window.colsGridTemplate = window.SBEditor.colsGridTemplate;
  window.galleryTemplate = window.SBEditor.galleryTemplate;
  window.cardsNormalizeItem = window.SBEditor.cardsNormalizeItem;

  window.saveBlockOrder = window.SBEditor.saveBlockOrder;
  window.initBlockDnD = window.SBEditor.initBlockDnD;
  window.renderBlocks = window.SBEditor.renderBlocks;

  window.SECTION_PRESETS = window.SBEditor.SECTION_PRESETS;
  window.sectionPresetOptions = window.SBEditor.sectionPresetOptions;
  window.applySectionPresetToForm = window.SBEditor.applySectionPresetToForm;
  window.createBlockAfterSection = window.SBEditor.createBlockAfterSection;
  window.quickAddHeadingAfterSection = window.SBEditor.quickAddHeadingAfterSection;
  window.quickAddTextAfterSection = window.SBEditor.quickAddTextAfterSection;
  window.quickAddButtonAfterSection = window.SBEditor.quickAddButtonAfterSection;
  window.quickAddCardsAfterSection = window.SBEditor.quickAddCardsAfterSection;
  window.addSectionBlock = window.SBEditor.addSectionBlock;
  window.editSectionBlock = window.SBEditor.editSectionBlock;
};


---

2. В editor.init.js добавь синхронизацию bridge

Найди:

window.SBEditor.setState(stPatch);

Сразу после неё вставь:

window.SBEditor.syncLegacyBridge();


---

3. И в loadBlocks() после обновления allPageBlocks тоже синхронизируй

Найди внутри window.SBEditor.loadBlocks = async function () { ... }:

st.allPageBlocks = Array.isArray(res.blocks) ? res.blocks.slice() : [];

Сразу после этого добавь:

window.SBEditor.syncLegacyBridge();


---

4. Подключи bridge-файл в editor.php

После:

<script src="/local/sitebuilder/assets/js/editor.blocks.js?v=1"></script>

добавь:

<script src="/local/sitebuilder/assets/js/editor.bridge.js?v=1"></script>

Итоговый порядок подключения должен быть такой:

<script src="/local/sitebuilder/assets/js/editor.core.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.api.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.sections.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.dnd.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.blocks.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.bridge.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.init.js?v=1"></script>


---

Зачем это нужно

После этого можно будет вынести большой блок диалогов из editor.php почти “как есть”, потому что старые имена (pageId, api, notify, fileDownloadUrl и т.д.) будут доступны и снаружи.

Это сильно снижает риск.

Следующий шаг

После bridge уже можно выносить:

addTextBlock

addImageBlock

addButtonBlock

addHeadingBlock

addCols2Block

addSpacerBlock

openGalleryDialog

openCardDialog

edit...Block


Если хочешь, следующим сообщением я дам уже готовый editor.dialogs.js под текущий проект через этот bridge.