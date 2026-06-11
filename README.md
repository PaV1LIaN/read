Отлично. Тогда следующий безопасный шаг: вынесем JS/CSS таблицы внутрь компонента table, как у disk.

То есть будет так:

/local/sitebuilder/components/table/
    render.php
    edit.js
    view.js
    styles.css

Старые файлы пока не удаляем, чтобы был откат.


---

1. Создай файлы компонента таблицы

Создай:

/local/sitebuilder/components/table/edit.js
/local/sitebuilder/components/table/view.js
/local/sitebuilder/components/table/styles.css


---

2. Скопируй текущие JS-файлы

Скопируй содержимое:

/local/sitebuilder/assets/public/table-edit.js

в:

/local/sitebuilder/components/table/edit.js

Скопируй содержимое:

/local/sitebuilder/assets/public/table-view.js

в:

/local/sitebuilder/components/table/view.js


---

3. В styles.css пока добавь только подключение таблицы

Файл:

/local/sitebuilder/components/table/styles.css

Вставь туда пока так:

/* =========================================================
   SiteBuilder component: table
   ========================================================= */

/* Сюда будем постепенно переносить стили таблицы из assets/public/public.css */

Пока стили остаются в public.css, чтобы ничего не сломать.


---

4. Подключи новые файлы в public_page.php

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди подключение старого файла:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-view.js"></script>

Замени на:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/table/styles.css">
<script src="<?= sb_public_h($basePath) ?>/components/table/view.js"></script>

Теперь найди старое подключение:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js"></script>

Замени на:

<script src="<?= sb_public_h($basePath) ?>/components/table/edit.js"></script>

Должно получиться примерно так:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/components/table/styles.css">
<script src="<?= sb_public_h($basePath) ?>/components/table/view.js"></script>

<?php if ($isPublicEditMode): ?>
    <script>
        window.SB_PUBLIC_EDIT_CONFIG = <?= json_encode([
            'apiUrl' => $basePath . '/api.php',
            'sessid' => bitrix_sessid(),
        ], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES) ?>;
    </script>

    <script src="<?= sb_public_h($basePath) ?>/components/table/edit.js"></script>
<?php endif; ?>


---

5. Проверь

Открой public-страницу без edit=Y.

В консоли должно быть:

window.SB_TABLE_VIEW_LOADED

Должно вернуть:

v1-public-pagination

Открой с edit=Y.

В консоли должно быть:

window.SB_TABLE_EDIT_LOADED

Должно вернуть:

v12-stable-types-pagination


---

6. Старые файлы пока не удаляй

Пока оставь:

/local/sitebuilder/assets/public/table-edit.js
/local/sitebuilder/assets/public/table-view.js

После проверки их можно будет удалить или оставить как резерв.


---

7. Что получится

Было:

assets/public/table-edit.js
assets/public/table-view.js
views/blocks/table.php

Стало:

components/table/render.php
components/table/edit.js
components/table/view.js
components/table/styles.css

Следующим шагом уже можно будет сделать автоматическое подключение CSS/JS компонентов, чтобы public_page.php сам подключал components/table/edit.js, components/disk/script.js и т.д. только если компонент реально есть на странице.