Идём дальше. Сейчас самый правильный шаг — подключить editor.dialogs.js и аккуратно убрать дубли из editor.php, но без резкого сноса всего inline-кода.

1. Подключение файлов в editor.php

Перед </body> поставь такой порядок:

<script>
  window.SB_EDITOR_SITE_ID = <?= (int)$siteId ?>;
  window.SB_EDITOR_PAGE_ID = <?= (int)$pageId ?>;
</script>

<script src="/local/sitebuilder/assets/js/editor.core.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.api.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.sections.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.dnd.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.blocks.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.bridge.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.dialogs.js?v=1"></script>
<script src="/local/sitebuilder/assets/js/editor.init.js?v=1"></script>


---

2. Что уже можно удалить из inline JS в editor.php

Если ты уже создал и заполнил:

editor.core.js

editor.api.js

editor.sections.js

editor.dnd.js

editor.blocks.js

editor.bridge.js

editor.dialogs.js

editor.init.js


то из editor.php уже можно убрать такие функции:

helpers/api

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


blocks/render

buildBlockShell

renderBlocks


dnd

saveBlockOrder

initBlockDnD


section

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


dialogs

saveTemplateFromPage

applyTemplateToPage

openSectionsLibrary

openCardsBuilderDialog

addTextBlock

addImageBlock

addButtonBlock

addHeadingBlock

addCols2Block

addGalleryBlock

addSpacerBlock

addCardBlock

addCardsBlock

editTextBlock

editImageBlock

editButtonBlock

editHeadingBlock

editCols2Block

editGalleryBlock

editSpacerBlock

editCardBlock

editCardsBlock



---

3. Что пока оставить в editor.php

Пока оставь:

HTML-разметку страницы

кнопки и контейнеры DOM

PHP-переменные $siteId, $pageId

если есть специфические куски Bitrix UI, которые ещё не вынесены

всё, что связано с версткой страницы, но не с логикой JS



---

4. Самый безопасный способ чистки

Не удаляй всё сразу одним махом.

Делай так:

1. Сначала подключи все .js


2. Проверь, что редактор открывается без ошибок


3. Потом удаляй из inline JS только один логический блок


4. Обновляй страницу и проверяй консоль



Лучший порядок удаления:

1. helpers/api


2. section


3. dnd


4. render


5. dialogs


6. старый BX.ready(...) хвост




---

5. Что проверить после подключения

Проверь:

редактор открывается

список блоков загружается

+ Section работает

редактирование обычных блоков работает

перетаскивание работает

поиск по блокам работает

кнопки + Heading / + Text / + Button / + Cards у section работают



---

6. Самый логичный следующий шаг

После подключения внешних файлов и удаления дублей следующим этапом надо будет:

почистить editor.php до состояния “шаблон + подключения”

То есть оставить в нём только:

PHP-часть

HTML

подключение CSS/JS


А весь JS вынести наружу.

Если хочешь, следующим сообщением я дам тебе чек-лист удаления по строкам/блокам, то есть в каком порядке вырезать куски из editor.php, чтобы ничего не сломать.