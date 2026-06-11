Отлично. Следующий этап лучше делать так:

Этап 2: ограничение количества строк + пагинация
Этап 3: сортировка по столбцам

Сейчас делаем лимит строк и пагинацию, чтобы не трогать сразу сортировку и не сломать типы/формулы, которые только стабилизировали.


---

1. Правим table.php

Файл:

/local/sitebuilder/views/blocks/table.php

1.1. После $rows добавь настройки

Найди сверху:

$columns = is_array($content['columns'] ?? null) ? $content['columns'] : [];
$rows = is_array($content['rows'] ?? null) ? $content['rows'] : [];

Сразу после добавь:

$settings = is_array($content['settings'] ?? null) ? $content['settings'] : [];

$maxRows = (int)($settings['maxRows'] ?? 0);
$pageSize = (int)($settings['pageSize'] ?? 10);
$paginationEnabled = !empty($settings['pagination']);

if ($maxRows < 0) {
    $maxRows = 0;
}

if ($pageSize < 1) {
    $pageSize = 10;
}

if ($pageSize > 200) {
    $pageSize = 200;
}


---

1.2. В $tableContent добавь settings

Найди:

$tableContent = [
    'title' => $title !== '' ? $title : 'Таблица',
    'columns' => $columns,
    'rows' => $normalizedRows,
];

Замени на:

$tableContent = [
    'title' => $title !== '' ? $title : 'Таблица',
    'columns' => $columns,
    'rows' => $normalizedRows,
    'settings' => [
        'maxRows' => $maxRows,
        'pageSize' => $pageSize,
        'pagination' => $paginationEnabled,
        'currentPage' => 1,
    ],
];


---

1.3. Добавь настройки в панель редактирования

Найди в editbar блок:

<label class="sb-public-table-editbar__label">
    Название таблицы
    <input
        class="sb-public-table-title-input"
        type="text"
        value="<?= sb_public_h($title !== '' ? $title : 'Таблица') ?>"
        data-table-title-input
    >
</label>

Сразу после него вставь:

<div class="sb-public-table-settings">
    <label class="sb-public-table-settings__field">
        Макс. строк
        <input
            type="number"
            min="0"
            step="1"
            value="<?= (int)$maxRows ?>"
            placeholder="0 = без лимита"
            data-table-max-rows
        >
    </label>

    <label class="sb-public-table-settings__field">
        Строк на странице
        <input
            type="number"
            min="1"
            max="200"
            step="1"
            value="<?= (int)$pageSize ?>"
            data-table-page-size
        >
    </label>

    <label class="sb-public-table-settings__check">
        <input
            type="checkbox"
            data-table-pagination-enabled
            <?= $paginationEnabled ? 'checked' : '' ?>
        >
        Пагинация
    </label>
</div>


---

1.4. После таблицы добавь блок пагинации

Найди закрытие:

</table>

Сразу после </table>, но ещё внутри .sb-public-table-wrap, вставь:

<?php if ($isEditMode): ?>
    <div class="sb-public-table-pagination" data-table-pagination></div>
<?php endif; ?>

Должно быть примерно так:

</table>

        <?php if ($isEditMode): ?>
            <div class="sb-public-table-pagination" data-table-pagination></div>
        <?php endif; ?>
    </div>


---

2. Правим table-edit.js

Файл:

/local/sitebuilder/assets/public/table-edit.js

2.1. Добавь функции настроек

После функции:

function setDirty(root, isDirty) {

после её закрытия вставь:

function normalizeSettings(settings) {
    settings = settings && typeof settings === 'object' ? settings : {};

    var maxRows = Number(settings.maxRows || 0);
    var pageSize = Number(settings.pageSize || 10);
    var currentPage = Number(settings.currentPage || 1);

    if (!Number.isFinite(maxRows) || maxRows < 0) {
        maxRows = 0;
    }

    if (!Number.isFinite(pageSize) || pageSize < 1) {
        pageSize = 10;
    }

    if (pageSize > 200) {
        pageSize = 200;
    }

    if (!Number.isFinite(currentPage) || currentPage < 1) {
        currentPage = 1;
    }

    return {
        maxRows: Math.floor(maxRows),
        pageSize: Math.floor(pageSize),
        pagination: !!settings.pagination,
        currentPage: Math.floor(currentPage)
    };
}

function getSettingsFromDom(root, oldContent) {
    oldContent = oldContent || getContent(root);

    var oldSettings = normalizeSettings(oldContent.settings || {});

    var maxRowsInput = root.querySelector('[data-table-max-rows]');
    var pageSizeInput = root.querySelector('[data-table-page-size]');
    var paginationInput = root.querySelector('[data-table-pagination-enabled]');

    var maxRows = maxRowsInput ? Number(maxRowsInput.value || 0) : oldSettings.maxRows;
    var pageSize = pageSizeInput ? Number(pageSizeInput.value || 10) : oldSettings.pageSize;

    return normalizeSettings({
        maxRows: maxRows,
        pageSize: pageSize,
        pagination: paginationInput ? !!paginationInput.checked : oldSettings.pagination,
        currentPage: oldSettings.currentPage || 1
    });
}

function setSettingsToDom(root, settings) {
    settings = normalizeSettings(settings);

    var maxRowsInput = root.querySelector('[data-table-max-rows]');
    var pageSizeInput = root.querySelector('[data-table-page-size]');
    var paginationInput = root.querySelector('[data-table-pagination-enabled]');

    if (maxRowsInput) {
        maxRowsInput.value = String(settings.maxRows);
    }

    if (pageSizeInput) {
        pageSizeInput.value = String(settings.pageSize);
    }

    if (paginationInput) {
        paginationInput.checked = !!settings.pagination;
    }
}


---

2.2. В collectContentFromDom() добавь settings

Найди в конце функции:

return {
    title: titleInput ? String(titleInput.value || '').trim() || 'Таблица' : (oldContent.title || 'Таблица'),
    columns: columns,
    rows: rows
};

Замени на:

return {
    title: titleInput ? String(titleInput.value || '').trim() || 'Таблица' : (oldContent.title || 'Таблица'),
    columns: columns,
    rows: rows,
    settings: getSettingsFromDom(root, oldContent)
};


---

2.3. В renderTableFromContent() сохрани настройки в DOM

Найди внутри функции:

content.columns = columns;
content.rows = rows;

Сразу после добавь:

content.settings = normalizeSettings(content.settings || {});
setSettingsToDom(root, content.settings);


---

2.4. Добавь пагинацию

После функции:

function renumberRows(root) {

после её закрытия вставь:

function applyPagination(root) {
    var content = getContent(root);
    var settings = normalizeSettings(content.settings || {});
    var rows = Array.isArray(content.rows) ? content.rows : [];
    var tbodyRows = Array.prototype.slice.call(root.querySelectorAll('tbody tr[data-row-id]'));
    var paginationBox = root.querySelector('[data-table-pagination]');

    if (!paginationBox) {
        return;
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

    var info = document.createElement('span');
    info.className = 'sb-public-table-pagination__info';
    info.textContent = 'Страница ' + settings.currentPage + ' из ' + totalPages + ', строк: ' + rows.length;

    var prevBtn = document.createElement('button');
    prevBtn.type = 'button';
    prevBtn.textContent = 'Назад';
    prevBtn.disabled = settings.currentPage <= 1;
    prevBtn.setAttribute('data-table-page-prev', '');

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

function setTablePage(root, page) {
    var content = collectContentFromDom(root);
    content.settings = normalizeSettings(content.settings || {});
    content.settings.currentPage = page;

    setContent(root, content);
    applyPagination(root);
    setDirty(root, true);
}


---

2.5. В renderTableFromContent() вызывай пагинацию

В конце renderTableFromContent(root, sourceContent) найди:

applyWidths(root);
applyAllAligns(root);
renumberRows(root);
updateFormulaCells(root);

Замени на:

applyWidths(root);
applyAllAligns(root);
renumberRows(root);
updateFormulaCells(root);
applyPagination(root);


---

2.6. В initTable() тоже вызывай пагинацию

Найди:

setContent(root, content);
renderTableFromContent(root, content);
setDirty(root, false);

После renderTableFromContent(root, content); уже будет вызов пагинации, но можно оставить так. Дополнительно ничего не надо.


---

2.7. Ограничь добавление строк

Найди функцию:

function addRow(root) {

Внутри неё после:

var rows = Array.isArray(content.rows) ? content.rows : [];

вставь:

var settings = normalizeSettings(content.settings || {});

if (settings.maxRows > 0 && rows.length >= settings.maxRows) {
    alert('Достигнут лимит строк: ' + settings.maxRows);
    return;
}


---

2.8. Добавь обработку кнопок пагинации и настроек

В обработчик:

document.addEventListener('click', function (e) {

после:

var root = e.target.closest('[data-public-editable-table]');

if (!root) {
    return;
}

вставь:

if (e.target.closest('[data-table-page-prev]')) {
    e.preventDefault();
    e.stopImmediatePropagation();

    var prevContent = getContent(root);
    var prevSettings = normalizeSettings(prevContent.settings || {});
    setTablePage(root, prevSettings.currentPage - 1);
    return;
}

if (e.target.closest('[data-table-page-next]')) {
    e.preventDefault();
    e.stopImmediatePropagation();

    var nextContent = getContent(root);
    var nextSettings = normalizeSettings(nextContent.settings || {});
    setTablePage(root, nextSettings.currentPage + 1);
    return;
}

Теперь найди обработчик:

document.addEventListener('input', function (e) {

В список условий добавь:

e.target.matches('[data-table-max-rows]') ||
e.target.matches('[data-table-page-size]') ||

То есть часть должна быть так:

if (
    e.target.matches('[data-table-title-input]') ||
    e.target.matches('[data-table-max-rows]') ||
    e.target.matches('[data-table-page-size]') ||
    e.target.matches('[data-column-label]') ||

Внутри этого блока после:

updateFormulaCells(root);

добавь:

applyPagination(root);

Теперь найди обработчик:

document.addEventListener('change', function (e) {

В самое начало после открытия функции вставь:

var paginationCheckbox = e.target.closest('[data-table-pagination-enabled]');

if (paginationCheckbox) {
    var paginationRoot = paginationCheckbox.closest('[data-public-editable-table]');

    if (!paginationRoot) {
        return;
    }

    e.stopImmediatePropagation();

    var paginationContent = collectContentFromDom(paginationRoot);
    paginationContent.settings = normalizeSettings(paginationContent.settings || {});
    paginationContent.settings.currentPage = 1;

    setContent(paginationRoot, paginationContent);
    applyPagination(paginationRoot);
    setDirty(paginationRoot, true);
    return;
}


---

3. CSS

В конец:

/local/sitebuilder/assets/public/public.css

добавь:

/* =========================================================
   Public table settings and pagination
   ========================================================= */

.sb-public-table-settings {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 8px;
    align-items: end;
}

.sb-public-table-settings__field {
    display: grid;
    gap: 4px;
    min-width: 130px;
    color: #475569;
    font-size: 11px;
    font-weight: 900;
}

.sb-public-table-settings__field input {
    width: 100%;
    min-height: 32px;
    box-sizing: border-box;
    padding: 5px 8px;
    border: 1px solid rgba(148, 163, 184, .7);
    border-radius: 9px;
    background: #fff;
    color: #0f172a;
    font-size: 12px;
    font-weight: 800;
}

.sb-public-table-settings__check {
    min-height: 32px;
    display: inline-flex;
    align-items: center;
    gap: 6px;
    padding: 5px 9px;
    border-radius: 9px;
    background: #f8fafc;
    border: 1px solid rgba(148, 163, 184, .45);
    color: #475569;
    font-size: 12px;
    font-weight: 900;
}

.sb-public-table-pagination {
    display: flex;
    align-items: center;
    justify-content: flex-end;
    gap: 8px;
    margin-top: 10px;
}

.sb-public-table-pagination:empty {
    display: none;
}

.sb-public-table-pagination button {
    min-height: 30px;
    padding: 5px 10px;
    border: 1px solid rgba(148, 163, 184, .65);
    border-radius: 9px;
    background: #fff;
    color: #1e40af;
    font-size: 12px;
    font-weight: 900;
    cursor: pointer;
}

.sb-public-table-pagination button:disabled {
    opacity: .45;
    cursor: default;
}

.sb-public-table-pagination__info {
    color: #475569;
    font-size: 12px;
    font-weight: 900;
}


---

После этого проверь:

1. Включи “Пагинация”.
2. Поставь “Строк на странице” = 2.
3. Добавь 4-5 строк.
4. Должны появиться кнопки Назад / Вперёд.
5. Поставь “Макс. строк” = 5.
6. При попытке добавить 6-ю строку должно быть предупреждение.

Если это работает — следующим шагом добавим сортировку по столбцам с учётом типов text / number / date / formula.