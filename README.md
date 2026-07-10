Теперь по коду можно сказать точно.

20-pages.js и 00-core.js работают правильно. Менять их не нужно.

Функция api() отправляет запрос строго по адресу из:

var API_URL = config.apiUrl || (BASE_PATH + '/api.php');

А config.apiUrl задаётся в editor.php. Сейчас там указан другой адрес:

apiUrl: '<?= CUtil::JSEscape($basePath) ?>/api.php',

При этом рабочий API, который ты уже проверил, находится по адресу:

/local/sitebuilder/api/index.php

Что изменить

В самом низу файла:

/local/sitebuilder/editor.php

найди:

<script>
window.SB_EDITOR_CONFIG = {
    basePath: '<?= CUtil::JSEscape($basePath) ?>',
    apiUrl: '<?= CUtil::JSEscape($basePath) ?>/api.php',
    siteId: <?= (int)$siteId ?>,
    isBitrixAdmin: <?= $USER->IsAdmin() ? 'true' : 'false' ?>,
    sessid: '<?= CUtil::JSEscape(bitrix_sessid()) ?>'
};
</script>

Замени только строку apiUrl:

<script>
window.SB_EDITOR_CONFIG = {
    basePath: '<?= CUtil::JSEscape($basePath) ?>',
    apiUrl: '<?= CUtil::JSEscape($basePath) ?>/api/index.php',
    siteId: <?= (int)$siteId ?>,
    isBitrixAdmin: <?= $USER->IsAdmin() ? 'true' : 'false' ?>,
    sessid: '<?= CUtil::JSEscape(bitrix_sessid()) ?>'
};
</script>

То есть:

/api.php

заменяется на:

/api/index.php

Что не менять

В 20-pages.js оставить:

state.pages = Array.isArray(res.pages) ? res.pages : [];

и:

state.blocks = Array.isArray(res.blocks) ? res.blocks : [];

Они полностью соответствуют фактическому ответу API.

В 00-core.js также ничего менять не нужно. После исправления конфигурации эта строка:

var API_URL = config.apiUrl || (BASE_PATH + '/api.php');

получит адрес /local/sitebuilder/api/index.php из window.SB_EDITOR_CONFIG.

Проверка после изменения

Обнови редактор через Ctrl + F5, затем выполни в консоли:

console.log({
    configApiUrl: window.SB_EDITOR_CONFIG.apiUrl,
    actualApiUrl: API_URL,
    siteId: siteId
});

Должно получиться:

{
    configApiUrl: "/local/sitebuilder/api/index.php",
    actualApiUrl: "/local/sitebuilder/api/index.php",
    siteId: 13
}

Затем выполни:

loadPages().then(function () {
    console.log('Страницы:', state.pages);
});

Должны появиться две страницы:

Диск
Тест

После этого:

state.currentPageId = 31;

loadBlocks().then(function () {
    console.log('Блоки:', state.blocks);
});

Должны загрузиться блоки 80 и 81.

Единственное, чего нельзя подтвердить по присланному коду: существует ли отдельный файл /local/sitebuilder/api.php, который перенаправляет запросы в /api/index.php. Но поскольку проверенный рабочий endpoint — /api/index.php, редактор сейчас нужно направить именно на него.