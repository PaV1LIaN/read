Скрин понятный. Сейчас две проблемы:

1. В edit=Y настройки пагинации дублируются.
2. Без edit=Y пагинация не работает, потому что table-edit.js подключается только в режиме редактирования.

Для обычной страницы нужен отдельный лёгкий файл, например:

/local/sitebuilder/assets/public/table-view.js

Он будет отвечать только за просмотр: скрывать лишние строки и показывать Назад / Вперёд.


---

1. Исправь дубль настроек в table-edit.js

Файл:

/local/sitebuilder/assets/public/table-edit.js

Найди функцию:

function ensureSettingsControls(root) {

Внутри неё найди:

if (root.querySelector('[data-public-table-settings]')) {
    return;
}

Замени на:

if (
    root.querySelector('[data-public-table-settings]') ||
    root.querySelector('[data-table-max-rows]') ||
    root.querySelector('[data-table-page-size]') ||
    root.querySelector('[data-table-pagination-enabled]')
) {
    return;
}

Так JS больше не будет создавать второй блок настроек, если он уже есть из table.php.


---

2. В table.php добавь атрибуты для просмотра

Файл:

/local/sitebuilder/views/blocks/table.php

Найди начало секции:

<section
    class="sb-block sb-block--table<?= $isEditMode ? ' is-public-editable-table' : '' ?>"

Замени на:

<section
    class="sb-block sb-block--table<?= $isEditMode ? ' is-public-editable-table' : '' ?>"
    data-public-table-view
    data-table-view-pagination="<?= $paginationEnabled ? '1' : '0' ?>"
    data-table-view-page-size="<?= (int)$pageSize ?>"

То есть атрибуты data-public-table-view, data-table-view-pagination, data-table-view-page-size должны быть всегда, даже без edit=Y.


---

3. В table.php поправь блок настроек

Найди:

<div class="sb-public-table-settings">

Замени на:

<div class="sb-public-table-settings" data-public-table-settings>

Это тоже уберёт дублирование настроек.


---

4. В table.php контейнер пагинации должен быть всегда

После:

</table>

должно быть:

<div class="sb-public-table-pagination" data-table-pagination></div>

Не оборачивай его в:

<?php if ($isEditMode): ?>

Нужно именно так:

</table>

        <div class="sb-public-table-pagination" data-table-pagination></div>
    </div>
</section>


---

5. Создай файл table-view.js

Файл:

/local/sitebuilder/assets/public/table-view.js

Вставь:

(function () {
    window.SB_TABLE_VIEW_LOADED = 'v1-public-pagination';

    function getPageSize(root) {
        var value = Number(root.getAttribute('data-table-view-page-size') || 10);

        if (!Number.isFinite(value) || value < 1) {
            value = 10;
        }

        if (value > 200) {
            value = 200;
        }

        return Math.floor(value);
    }

    function isPaginationEnabled(root) {
        return String(root.getAttribute('data-table-view-pagination') || '') === '1';
    }

    function ensurePaginationBox(root) {
        var box = root.querySelector('[data-table-pagination]');

        if (box) {
            return box;
        }

        box = document.createElement('div');
        box.className = 'sb-public-table-pagination';
        box.setAttribute('data-table-pagination', '');

        var wrap = root.querySelector('.sb-public-table-wrap');

        if (wrap) {
            wrap.appendChild(box);
        } else {
            root.appendChild(box);
        }

        return box;
    }

    function getRows(root) {
        return Array.prototype.slice.call(root.querySelectorAll('tbody tr[data-row-id]'));
    }

    function renderPagination(root) {
        if (root.hasAttribute('data-public-editable-table')) {
            return;
        }

        var enabled = isPaginationEnabled(root);
        var rows = getRows(root);
        var paginationBox = ensurePaginationBox(root);

        if (!enabled || rows.length === 0) {
            paginationBox.innerHTML = '';

            rows.forEach(function (tr) {
                tr.style.display = '';
            });

            return;
        }

        var pageSize = getPageSize(root);
        var totalPages = Math.max(1, Math.ceil(rows.length / pageSize));
        var currentPage = Number(root.getAttribute('data-table-view-current-page') || 1);

        if (!Number.isFinite(currentPage) || currentPage < 1) {
            currentPage = 1;
        }

        if (currentPage > totalPages) {
            currentPage = totalPages;
        }

        root.setAttribute('data-table-view-current-page', String(currentPage));

        var start = (currentPage - 1) * pageSize;
        var end = start + pageSize;

        rows.forEach(function (tr, index) {
            tr.style.display = index >= start && index < end ? '' : 'none';
        });

        paginationBox.innerHTML = '';

        if (totalPages <= 1) {
            return;
        }

        var prevBtn = document.createElement('button');
        prevBtn.type = 'button';
        prevBtn.textContent = 'Назад';
        prevBtn.disabled = currentPage <= 1;
        prevBtn.setAttribute('data-table-view-page-prev', '');

        var info = document.createElement('span');
        info.className = 'sb-public-table-pagination__info';
        info.textContent = 'Страница ' + currentPage + ' из ' + totalPages + ', строк: ' + rows.length;

        var nextBtn = document.createElement('button');
        nextBtn.type = 'button';
        nextBtn.textContent = 'Вперёд';
        nextBtn.disabled = currentPage >= totalPages;
        nextBtn.setAttribute('data-table-view-page-next', '');

        paginationBox.appendChild(prevBtn);
        paginationBox.appendChild(info);
        paginationBox.appendChild(nextBtn);
    }

    function initAll() {
        document.querySelectorAll('[data-public-table-view]').forEach(function (root) {
            renderPagination(root);
        });
    }

    document.addEventListener('click', function (e) {
        var prevBtn = e.target.closest('[data-table-view-page-prev]');
        var nextBtn = e.target.closest('[data-table-view-page-next]');

        if (!prevBtn && !nextBtn) {
            return;
        }

        var root = e.target.closest('[data-public-table-view]');

        if (!root) {
            return;
        }

        e.preventDefault();

        var currentPage = Number(root.getAttribute('data-table-view-current-page') || 1);

        if (!Number.isFinite(currentPage) || currentPage < 1) {
            currentPage = 1;
        }

        if (prevBtn) {
            currentPage -= 1;
        }

        if (nextBtn) {
            currentPage += 1;
        }

        root.setAttribute('data-table-view-current-page', String(currentPage));

        renderPagination(root);
    }, true);

    if (document.readyState === 'loading') {
        document.addEventListener('DOMContentLoaded', initAll);
    } else {
        initAll();
    }
})();


---

6. Подключи table-view.js всегда

Файл:

/local/sitebuilder/views/layout/public_page.php

Найди, где подключаются public-скрипты.

Добавь вне условия edit=Y, чтобы файл грузился и на обычной странице:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-view.js"></script>

Например:

<script src="<?= sb_public_h($basePath) ?>/assets/public/table-view.js"></script>

<?php if ($isPublicEditMode): ?>
    <script>
        window.SB_PUBLIC_EDIT_CONFIG = <?= json_encode([
            'apiUrl' => $basePath . '/api.php',
            'sessid' => bitrix_sessid(),
        ], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES) ?>;
    </script>

    <script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js"></script>
<?php endif; ?>


---

После этого:

1. Открой edit=Y.
2. Включи пагинацию.
3. Строк на странице = 2.
4. Нажми “Сохранить изменения”.
5. Открой без edit=Y.

На обычной странице должно показывать только 2 строки и кнопки Назад / Вперёд.