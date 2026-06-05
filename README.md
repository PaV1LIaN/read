Понял. Сейчас надо не гадать, а за 30 секунд понять, что именно не подхватилось: HTML или JS.

Открой public-страницу с edit=Y, нажми F12 → Console и выполни по очереди:

window.SB_TABLE_EDIT_LOADED

Должно быть:

"v7-delegated-columns-align"

Потом:

document.querySelectorAll('[data-table-add-column]').length

Должно быть больше 0.

Потом:

document.querySelectorAll('[data-column-align]').length

Должно быть больше 0.

Потом:

document.querySelectorAll('[data-public-editable-table]').length

Должно быть больше 0.

Что означают результаты

Если window.SB_TABLE_EDIT_LOADED показывает не v7-delegated-columns-align, значит подключается старый table-edit.js. Тогда в файле:

/local/sitebuilder/views/layout/public_page.php

поставь версию повыше:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js?v=100"></script>

и сделай Ctrl + F5.

Если [data-table-add-column] равно 0, значит не обновился table.php. То есть на странице нет кнопки с нужным атрибутом. Тогда проверь, что ты заменил именно:

/local/sitebuilder/views/blocks/table.php

и что в нём есть:

<button class="sb-public-table-editbar__btn sb-public-table-editbar__btn--light" type="button" data-table-add-column>
    + Столбец
</button>

Если [data-column-align] равно 0, значит в table.php нет select для выравнивания. Должно быть:

<select class="sb-public-table-align-select" data-column-align>
    <option value="left"<?= $column['align'] === 'left' ? ' selected' : '' ?>>Слева</option>
    <option value="center"<?= $column['align'] === 'center' ? ' selected' : '' ?>>Центр</option>
    <option value="right"<?= $column['align'] === 'right' ? ' selected' : '' ?>>Справа</option>
</select>

Если все значения нормальные, но всё равно не работает, вставь в консоль вот это:

document.addEventListener('click', function(e) {
    if (e.target.closest('[data-table-add-column]')) {
        console.log('CLICK ADD COLUMN OK');
    }
}, true);

document.addEventListener('change', function(e) {
    if (e.target.closest('[data-column-align]')) {
        console.log('CHANGE ALIGN OK', e.target.value);
    }
}, true);

После этого нажми + Столбец и поменяй выравнивание. Если сообщения в консоли не появляются — клик перехватывает другой слой/элемент. Если появляются — проблема уже внутри функции добавления.

Пришли мне результаты этих 4 команд:

window.SB_TABLE_EDIT_LOADED
document.querySelectorAll('[data-table-add-column]').length
document.querySelectorAll('[data-column-align]').length
document.querySelectorAll('[data-public-editable-table]').length