Да, сейчас делаем безопасный первый шаг: выносим большой JavaScript из editor.php в отдельный файл.

Шаг 0. Сделай backup

Перед правками сохрани текущий файл:

cp /srv/bx/docroot/local/sitebuilder/editor.php /srv/bx/docroot/local/sitebuilder/editor.php.backup_before_js_split


---

Шаг 1. Создай новый файл

Создай файл:

/local/sitebuilder/assets/admin/editor.js


---

Шаг 2. Перенеси JS из editor.php

В твоём editor.php найди вот этот большой блок:

<script src="/bitrix/js/main/core/core.js"></script>
<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var siteId = <?= (int)$siteId ?>;
    var IS_BITRIX_ADMIN = <?= $USER->IsAdmin() ? 'true' : 'false' ?>;

    ...
})();
</script>

Из него в новый файл editor.js нужно перенести только содержимое второго <script>, то есть начиная с:

(function () {

и заканчивая:

})();

<script> и </script> в editor.js писать не надо.


---

Шаг 3. В начале editor.js замени первые строки

В новом editor.js в самом начале сейчас будет так:

(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var siteId = <?= (int)$siteId ?>;
    var IS_BITRIX_ADMIN = <?= $USER->IsAdmin() ? 'true' : 'false' ?>;

Замени на:

(function () {
    var config = window.SB_EDITOR_CONFIG || {};

    var BASE_PATH = config.basePath || '';
    var API_URL = config.apiUrl || (BASE_PATH + '/api.php');
    var siteId = Number(config.siteId || 0);
    var IS_BITRIX_ADMIN = !!config.isBitrixAdmin;


---

Шаг 4. В editor.js замени функцию getSessid()

Найди:

function getSessid() {
    if (window.BX && typeof BX.bitrix_sessid === 'function') {
        return BX.bitrix_sessid();
    }

    return '<?= CUtil::JSEscape(bitrix_sessid()) ?>';
}

Замени на:

function getSessid() {
    if (window.BX && typeof BX.bitrix_sessid === 'function') {
        return BX.bitrix_sessid();
    }

    return config.sessid || '';
}


---

Шаг 5. В editor.php вместо большого JS оставь подключение

В editor.php внизу вместо этого:

<script src="/bitrix/js/main/core/core.js"></script>
<script>
(function () {
    ...
})();
</script>

поставь:

<script>
window.SB_EDITOR_CONFIG = {
    basePath: '<?= CUtil::JSEscape($basePath) ?>',
    apiUrl: '<?= CUtil::JSEscape($basePath) ?>/api.php',
    siteId: <?= (int)$siteId ?>,
    isBitrixAdmin: <?= $USER->IsAdmin() ? 'true' : 'false' ?>,
    sessid: '<?= CUtil::JSEscape(bitrix_sessid()) ?>'
};
</script>

<script src="/bitrix/js/main/core/core.js"></script>
<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.js?v=1"></script>


---

Шаг 6. Проверь

Открой:

/local/sitebuilder/editor.php?siteId=ID_САЙТА

И проверь:

1. Страницы слева загружаются.
2. Блоки по центру загружаются.
3. Сохранение страницы работает.
4. Сохранение блока работает.
5. Диск-блок редактируется.
6. Права пользователей открываются.
7. Кнопка “Сохранить как шаблон” работает.

Если всё работает — отлично, можно идти дальше и добавлять секции уже в editor.js, а не в огромный editor.php.


---

Что пришлёшь мне после этого

После выноса пришли мне новый файл:

/local/sitebuilder/assets/admin/editor.js

И я уже в него аккуратно добавлю функционал секций страницы.