В самом editor.php данные страниц и блоков не разбираются — это делают подключённые JavaScript-файлы. Но здесь есть одна важная ошибка.

1. Исправь адрес API в editor.php

В самом низу файла сейчас:

apiUrl: '<?= CUtil::JSEscape($basePath) ?>/api.php',

А рабочий API, который ты проверил в консоли:

/local/sitebuilder/api/index.php

Замени блок конфигурации на:

<script>
window.SB_EDITOR_CONFIG = {
    basePath: '<?= CUtil::JSEscape($basePath) ?>',
    apiUrl: '<?= CUtil::JSEscape($basePath) ?>/api/index.php',
    siteId: <?= (int)$siteId ?>,
    isBitrixAdmin: <?= $USER->IsAdmin() ? 'true' : 'false' ?>,
    sessid: '<?= CUtil::JSEscape(bitrix_sessid()) ?>'
};
</script>

То есть меняется только:

/api.php

на:

/api/index.php


---

2. Страницы исправляются не здесь, а в файле

/local/sitebuilder/assets/admin/editor/20-pages.js

Найди там функцию вроде:

loadPages

или строку:

data.pages

или:

result.data.pages

Было примерно так:

const pages = result.data?.pages || [];

Замени на:

const pages = Array.isArray(result.pages)
    ? result.pages
    : [];

Более безопасный вариант, который поддерживает оба формата API:

const pages = Array.isArray(result.pages)
    ? result.pages
    : Array.isArray(result.data?.pages)
        ? result.data.pages
        : [];

Затем сохрани страницы в общее состояние редактора. Например:

SB.state.pages = pages;

или, в зависимости от названия объекта у тебя:

state.pages = pages;

Не меняй название SB или state, пока не увидишь, какое используется в самом файле.


---

3. Блоки исправляются в файле

/local/sitebuilder/assets/admin/editor/30-blocks.js

Найди:

result.data.blocks

или:

data.blocks

Было:

const blocks = result.data?.blocks || [];

Замени на:

const blocks = Array.isArray(result.blocks)
    ? result.blocks
    : [];

Безопасный вариант:

const blocks = Array.isArray(result.blocks)
    ? result.blocks
    : Array.isArray(result.data?.blocks)
        ? result.data.blocks
        : [];

После этого блоки нужно записать в состояние:

SB.state.blocks = blocks;

либо:

state.blocks = blocks;

Используй название, которое уже применяется в 30-blocks.js.


---

4. Обнови версии JavaScript-файлов

Чтобы браузер не загрузил старый JavaScript из кэша, в editor.php внизу замени v=2 на v=3:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/00-core.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/10-sections.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/20-pages.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/30-blocks.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/40-access.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/50-template.js?v=3"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor/60-events.js?v=3"></script>

Также CSS можно обновить:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.css?v=3">

Что конкретно сделать сейчас

В editor.php обязательно исправь:

apiUrl: '<?= CUtil::JSEscape($basePath) ?>/api/index.php',

Затем нужны изменения в двух файлах:

/local/sitebuilder/assets/admin/editor/20-pages.js
/local/sitebuilder/assets/admin/editor/30-blocks.js

Потому что именно там находятся page.list и block.list. В присланном editor.php строк result.data.pages и result.data.blocks нет.