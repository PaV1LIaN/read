Понял. Скорее всего причина простая: контейнер пагинации не появился в HTML или он добавлен только при edit=Y. Сделаем надёжно: пагинация будет создаваться сама через JS, даже если ты забыл добавить <div data-table-pagination> в table.php.

1. В table.php добавь контейнер пагинации всегда

Файл:

/local/sitebuilder/views/blocks/table.php

Найди:

</table>

Сразу после него вставь:

<div class="sb-public-table-pagination" data-table-pagination></div>

Должно быть так:

</table>

        <div class="sb-public-table-pagination" data-table-pagination></div>
    </div>
</section>

Не оборачивай в if ($isEditMode), пусть блок будет всегда.


---

2. В table-edit.js замени функцию applyPagination

Файл:

/local/sitebuilder/assets/public/table-edit.js

Найди функцию:

function applyPagination(root) {

Полностью замени её на:

function applyPagination(root) {
    var content = getContent(root);
    var settings = normalizeSettings(content.settings || {});
    var rows = Array.isArray(content.rows) ? content.rows : [];
    var table = root.querySelector('.sb-public-table');

    if (!table) {
        return;
    }

    var tbodyRows = Array.prototype.slice.call(root.querySelectorAll('tbody tr[data-row-id]'));
    var paginationBox = root.querySelector('[data-table-pagination]');

    if (!paginationBox) {
        paginationBox = document.createElement('div');
        paginationBox.className = 'sb-public-table-pagination';
        paginationBox.setAttribute('data-table-pagination', '');

        var wrap = root.querySelector('.sb-public-table-wrap');

        if (wrap) {
            wrap.appendChild(paginationBox);
        } else {
            root.appendChild(paginationBox);
        }
    }

    if (!settings.pagination || !rows.length) {
        paginationBox.innerHTML = '';

        tbodyRows.forEach(function (tr) {
            tr.style.display = '';
        });

        return;
    }

    var pageSize = settings.pageSize || 10;
    var totalPages = Math.max(1, Math.ceil(rows.length / pageSize));

    if (settings.currentPage > totalPages) {
        settings.currentPage = totalPages;
    }

    if (settings.currentPage < 1) {
        settings.currentPage = 1;
    }

    var start = (settings.currentPage - 1) * pageSize;
    var end = start + pageSize;

    tbodyRows.forEach(function (tr, index) {
        tr.style.display = index >= start && index < end ? '' : 'none';

        var num = tr.querySelector('.sb-public-table-row-num');

        if (num) {
            num.textContent = String(index + 1);
        }
    });

    paginationBox.innerHTML = '';

    if (totalPages <= 1) {
        content.settings = settings;
        setContent(root, content);
        return;
    }

    var prevBtn = document.createElement('button');
    prevBtn.type = 'button';
    prevBtn.textContent = 'Назад';
    prevBtn.disabled = settings.currentPage <= 1;
    prevBtn.setAttribute('data-table-page-prev', '');

    var info = document.createElement('span');
    info.className = 'sb-public-table-pagination__info';
    info.textContent = 'Страница ' + settings.currentPage + ' из ' + totalPages + ', строк: ' + rows.length;

    var nextBtn = document.createElement('button');
    nextBtn.type = 'button';
    nextBtn.textContent = 'Вперёд';
    nextBtn.disabled = settings.currentPage >= totalPages;
    nextBtn.setAttribute('data-table-page-next', '');

    paginationBox.appendChild(prevBtn);
    paginationBox.appendChild(info);
    paginationBox.appendChild(nextBtn);

    content.settings = settings;
    setContent(root, content);
}


---

3. Проверь, что в renderTableFromContent вызывается пагинация

В конце функции:

function renderTableFromContent(root, sourceContent) {

должно быть:

applyWidths(root);
applyAllAligns(root);
renumberRows(root);
updateFormulaCells(root);
applyPagination(root);

Если у тебя сейчас так:

applyWidths(root);
applyAllAligns(root);
renumberRows(root);
updateFormulaCells(root);

добавь строку:

applyPagination(root);


---

4. Проверь обработку кнопок Назад / Вперёд

В обработчике:

document.addEventListener('click', function (e) {

после этого куска:

var root = e.target.closest('[data-public-editable-table]');

if (!root) {
    return;
}

должно быть:

if (e.target.closest('[data-table-page-prev]')) {
    e.preventDefault();
    e.stopImmediatePropagation();

    var prevContent = getContent(root);
    var prevSettings = normalizeSettings(prevContent.settings || {});

    prevSettings.currentPage = prevSettings.currentPage - 1;
    prevContent.settings = prevSettings;

    setContent(root, prevContent);
    applyPagination(root);
    setDirty(root, true);
    return;
}

if (e.target.closest('[data-table-page-next]')) {
    e.preventDefault();
    e.stopImmediatePropagation();

    var nextContent = getContent(root);
    var nextSettings = normalizeSettings(nextContent.settings || {});

    nextSettings.currentPage = nextSettings.currentPage + 1;
    nextContent.settings = nextSettings;

    setContent(root, nextContent);
    applyPagination(root);
    setDirty(root, true);
    return;
}


---

5. Важно

Пагинация появится только если:

1. включена галочка “Пагинация”;
2. строк больше, чем “Строк на странице”.

Например:

Пагинация: включена
Строк на странице: 2
Строк в таблице: 5

Тогда появятся кнопки:

Назад | Страница 1 из 3 | Вперёд

После правок сделай Ctrl + F5, включи пагинацию, поставь Строк на странице = 1 и добавь хотя бы 3 строки.