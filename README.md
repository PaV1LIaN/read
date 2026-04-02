Проверил текущий проект и собрал заново оба файла.

Готовые файлы:

editor.dialogs.js

editor.init.js


Что делать сейчас:

1. Замени ими файлы:

/local/sitebuilder/assets/js/editor.dialogs.js

/local/sitebuilder/assets/js/editor.init.js



2. Подключения в editor.php оставь в таком порядке:



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

Важно:

эти новые файлы собраны так, чтобы работать с уже существующими editor.core.js, editor.api.js, editor.sections.js, editor.dnd.js, editor.blocks.js, editor.bridge.js

в новом editor.dialogs.js я уже добавил недостающий helper для секций, который нужен библиотеке секций


После замены сначала просто открой редактор и проверь консоль. Если страница загрузится без JS-ошибок, следующим шагом уже можно чистить inline JS из editor.php.