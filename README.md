Да, начнём с изменения ширины столбцов прямо в public. Остальные изменения данных прямо на странице сделаем следующим шагом.

Сейчас добавим режим:

public.php?siteId=13&pageId=14&edit=Y

В этом режиме у таблицы появятся “ручки” на границах столбцов и кнопка Сохранить ширину.


---

1. Замени table.php

Файл:

/local/sitebuilder/views/blocks/table.php

Полностью замени на:

<?php
global $USER;

$title = trim((string)($content['title'] ?? ''));

$columns = is_array($content['columns'] ?? null) ? $content['columns'] : [];
$rows = is_array($content['rows'] ?? null) ? $content['rows'] : [];

if (empty($columns)) {
    return;
}

$isEditMode = (
    (string)($_GET['edit'] ?? '') === 'Y'
    && is_object($USER)
    && method_exists($USER, 'IsAuthorized')
    && $USER->IsAuthorized()
    && method_exists($USER, 'IsAdmin')
    && $USER->IsAdmin()
);

$columns = array_values(array_map(static function ($column, $index) {
    $id = trim((string)($column['id'] ?? ''));

    if ($id === '') {
        $id = 'col_' . ($index + 1);
    }

    $label = trim((string)($column['label'] ?? ''));

    if ($label === '') {
        $label = 'Столбец ' . ($index + 1);
    }

    $width = (int)($column['width'] ?? 0);

    if ($width < 40) {
        $width = 0;
    }

    if ($width > 1200) {
        $width = 1200;
    }

    return [
        'id' => $id,
        'label' => $label,
        'width' => $width,
    ];
}, $columns, array_keys($columns)));

$normalizedRows = [];

foreach ($rows as $rowIndex => $row) {
    $cells = is_array($row['cells'] ?? null) ? $row['cells'] : [];
    $rowId = trim((string)($row['id'] ?? ''));

    if ($rowId === '') {
        $rowId = 'row_' . ($rowIndex + 1);
    }

    $normalizedRows[] = [
        'id' => $rowId,
        'cells' => $cells,
    ];
}

$tableContent = [
    'title' => $title !== '' ? $title : 'Таблица',
    'columns' => $columns,
    'rows' => $normalizedRows,
];

$contentJson = json_encode($tableContent, JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);
$propsJson = json_encode(is_array($props ?? null) ? $props : [], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES);

$blockId = (int)($block['id'] ?? 0);
?>

<section
    class="sb-block sb-block--table<?= $isEditMode ? ' is-public-editable-table' : '' ?>"
    <?php if ($isEditMode): ?>
        data-public-editable-table
        data-block-id="<?= $blockId ?>"
        data-content="<?= sb_public_h((string)$contentJson) ?>"
        data-props="<?= sb_public_h((string)$propsJson) ?>"
    <?php endif; ?>
>
    <?php if ($title !== ''): ?>
        <h2 class="sb-public-table__title"><?= sb_public_h($title) ?></h2>
    <?php endif; ?>

    <?php if ($isEditMode): ?>
        <div class="sb-public-table-editbar">
            <div class="sb-public-table-editbar__text">
                Режим редактирования: можно менять ширину столбцов
            </div>

            <button class="sb-public-table-editbar__btn" type="button" data-table-save-widths>
                Сохранить ширину
            </button>
        </div>
    <?php endif; ?>

    <div class="sb-public-table-wrap">
        <table class="sb-public-table<?= $isEditMode ? ' sb-public-table--editable' : '' ?>">
            <colgroup>
                <?php foreach ($columns as $column): ?>
                    <?php
                    $style = '';

                    if ((int)$column['width'] > 0) {
                        $style = ' style="width:' . (int)$column['width'] . 'px;"';
                    }
                    ?>
                    <col data-column-id="<?= sb_public_h($column['id']) ?>"<?= $style ?>>
                <?php endforeach; ?>
            </colgroup>

            <thead>
                <tr>
                    <?php foreach ($columns as $column): ?>
                        <?php
                        $style = '';

                        if ((int)$column['width'] > 0) {
                            $style = ' style="width:' . (int)$column['width'] . 'px;"';
                        }
                        ?>
                        <th data-column-id="<?= sb_public_h($column['id']) ?>"<?= $style ?>>
                            <span class="sb-public-table__th-text"><?= sb_public_h($column['label']) ?></span>

                            <?php if ($isEditMode): ?>
                                <span class="sb-public-table-resizer" data-column-resizer></span>
                            <?php endif; ?>
                        </th>
                    <?php endforeach; ?>
                </tr>
            </thead>

            <tbody>
                <?php if (!empty($normalizedRows)): ?>
                    <?php foreach ($normalizedRows as $row): ?>
                        <?php $cells = is_array($row['cells'] ?? null) ? $row['cells'] : []; ?>
                        <tr data-row-id="<?= sb_public_h((string)$row['id']) ?>">
                            <?php foreach ($columns as $column): ?>
                                <td data-column-id="<?= sb_public_h($column['id']) ?>">
                                    <?= nl2br(sb_public_h((string)($cells[$column['id']] ?? ''))) ?>
                                </td>
                            <?php endforeach; ?>
                        </tr>
                    <?php endforeach; ?>
                <?php else: ?>
                    <tr>
                        <td colspan="<?= count($columns) ?>">Нет данных</td>
                    </tr>
                <?php endif; ?>
            </tbody>
        </table>
    </div>
</section>


---

2. Создай JS для public-редактирования таблиц

Создай файл:

/local/sitebuilder/assets/public/table-edit.js

Вставь:

(function () {
    var config = window.SB_PUBLIC_EDIT_CONFIG || {};
    var API_URL = config.apiUrl || '/local/sitebuilder/api.php';
    var sessid = config.sessid || '';

    function parseJson(value, fallback) {
        try {
            return JSON.parse(value || '');
        } catch (e) {
            return fallback;
        }
    }

    function setDirty(root, isDirty) {
        root.classList.toggle('is-dirty', !!isDirty);

        var btn = root.querySelector('[data-table-save-widths]');

        if (btn) {
            btn.textContent = isDirty ? 'Сохранить ширину *' : 'Сохранить ширину';
        }
    }

    function saveBlock(root) {
        var blockId = Number(root.getAttribute('data-block-id') || 0);
        var content = parseJson(root.getAttribute('data-content'), {});
        var props = parseJson(root.getAttribute('data-props'), {});

        if (!blockId) {
            alert('Не найден ID блока таблицы');
            return;
        }

        var formData = new FormData();

        formData.append('action', 'block.update');
        formData.append('sessid', sessid);
        formData.append('id', String(blockId));
        formData.append('content', JSON.stringify(content));
        formData.append('props', JSON.stringify(props || {}));

        var btn = root.querySelector('[data-table-save-widths]');

        if (btn) {
            btn.disabled = true;
            btn.textContent = 'Сохраняю...';
        }

        fetch(API_URL, {
            method: 'POST',
            body: formData,
            credentials: 'same-origin'
        })
            .then(function (response) {
                return response.json();
            })
            .then(function (res) {
                if (!res || !res.ok) {
                    throw new Error((res && (res.message || res.error)) || 'SAVE_ERROR');
                }

                root.setAttribute('data-content', JSON.stringify(content));
                setDirty(root, false);

                if (btn) {
                    btn.textContent = 'Сохранено';
                    setTimeout(function () {
                        btn.textContent = 'Сохранить ширину';
                    }, 1000);
                }
            })
            .catch(function (err) {
                console.error(err);
                alert('Не удалось сохранить ширину столбцов: ' + err.message);
                setDirty(root, true);
            })
            .finally(function () {
                if (btn) {
                    btn.disabled = false;
                }
            });
    }

    function initTable(root) {
        var table = root.querySelector('.sb-public-table');
        var content = parseJson(root.getAttribute('data-content'), {});

        if (!table || !content || !Array.isArray(content.columns)) {
            return;
        }

        var active = null;

        root.querySelectorAll('[data-column-resizer]').forEach(function (resizer) {
            resizer.addEventListener('mousedown', function (e) {
                e.preventDefault();

                var th = resizer.closest('th[data-column-id]');

                if (!th) {
                    return;
                }

                var columnId = String(th.getAttribute('data-column-id') || '');
                var col = table.querySelector('col[data-column-id="' + columnId + '"]');

                active = {
                    root: root,
                    table: table,
                    th: th,
                    col: col,
                    columnId: columnId,
                    startX: e.clientX,
                    startWidth: th.getBoundingClientRect().width
                };

                document.body.classList.add('sb-public-table-resizing');
            });
        });

        document.addEventListener('mousemove', function (e) {
            if (!active) {
                return;
            }

            var diff = e.clientX - active.startX;
            var newWidth = Math.round(active.startWidth + diff);

            if (newWidth < 80) {
                newWidth = 80;
            }

            if (newWidth > 1200) {
                newWidth = 1200;
            }

            active.th.style.width = newWidth + 'px';

            if (active.col) {
                active.col.style.width = newWidth + 'px';
            }

            content.columns = content.columns.map(function (column) {
                if (String(column.id) === active.columnId) {
                    column.width = newWidth;
                }

                return column;
            });

            root.setAttribute('data-content', JSON.stringify(content));
            setDirty(root, true);
        });

        document.addEventListener('mouseup', function () {
            if (!active) {
                return;
            }

            active = null;
            document.body.classList.remove('sb-public-table-resizing');
        });

        var saveBtn = root.querySelector('[data-table-save-widths]');

        if (saveBtn) {
            saveBtn.addEventListener('click', function () {
                saveBlock(root);
            });
        }
    }

    document.addEventListener('DOMContentLoaded', function () {
        document.querySelectorAll('[data-public-editable-table]').forEach(initTable);
    });
})();


---

3. Подключи JS в public_page.php

Файл:

/local/sitebuilder/public_page.php

Найди место внизу, где подключается disk script или перед </body>.

Вставь перед </body>:

<?php
global $USER;

$isPublicEditMode = (
    (string)($_GET['edit'] ?? '') === 'Y'
    && is_object($USER)
    && $USER->IsAuthorized()
    && $USER->IsAdmin()
);
?>

<?php if ($isPublicEditMode): ?>
    <script>
        window.SB_PUBLIC_EDIT_CONFIG = <?= json_encode([
            'apiUrl' => $basePath . '/api.php',
            'sessid' => bitrix_sessid(),
        ], JSON_UNESCAPED_UNICODE | JSON_UNESCAPED_SLASHES) ?>;
    </script>

    <script src="<?= sb_public_h($basePath) ?>/assets/public/table-edit.js?v=1"></script>
<?php endif; ?>


---

4. Добавь стили в public.css

Файл:

/local/sitebuilder/assets/public/public.css

В самый конец добавь:

/* =========================================================
   Public table edit mode
   ========================================================= */

.sb-public-table-editbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    margin: 0 0 12px;
    padding: 10px 12px;
    border: 1px dashed rgba(37, 99, 235, .45);
    border-radius: 14px;
    background: rgba(37, 99, 235, .06);
}

.sb-public-table-editbar__text {
    color: #1e40af;
    font-size: 13px;
    font-weight: 800;
}

.sb-public-table-editbar__btn {
    min-height: 34px;
    padding: 7px 12px;
    border: 0;
    border-radius: 10px;
    background: #2563eb;
    color: #fff;
    font-size: 13px;
    font-weight: 900;
    cursor: pointer;
}

.sb-public-table-editbar__btn:disabled {
    opacity: .65;
    cursor: wait;
}

.sb-block--table.is-dirty .sb-public-table-editbar {
    border-color: rgba(245, 158, 11, .75);
    background: rgba(245, 158, 11, .08);
}

.sb-public-table--editable {
    table-layout: fixed;
}

.sb-public-table--editable th {
    position: relative;
    user-select: none;
}

.sb-public-table__th-text {
    display: block;
    padding-right: 10px;
    overflow: hidden;
    text-overflow: ellipsis;
}

.sb-public-table-resizer {
    position: absolute;
    top: 0;
    right: -4px;
    width: 8px;
    height: 100%;
    cursor: col-resize;
    z-index: 5;
}

.sb-public-table-resizer::after {
    content: "";
    position: absolute;
    top: 8px;
    bottom: 8px;
    left: 3px;
    width: 2px;
    border-radius: 99px;
    background: rgba(37, 99, 235, .35);
    opacity: 0;
    transition: opacity .15s ease;
}

.sb-public-table-resizer:hover::after,
.sb-public-table-resizing .sb-public-table-resizer::after {
    opacity: 1;
}

body.sb-public-table-resizing {
    cursor: col-resize !important;
    user-select: none;
}


---

5. Обнови версию CSS в public_page.php

Найди подключение:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=12">

Поставь, например:

<link rel="stylesheet" href="<?= sb_public_h($basePath) ?>/assets/public/public.css?v=13">


---

Как проверить

Открой public-страницу так:

/local/sitebuilder/public.php?siteId=13&pageId=14&edit=Y

У таблицы должна появиться синяя панель:

Режим редактирования: можно менять ширину столбцов
[Сохранить ширину]

Наведи мышкой на границу между заголовками столбцов, потяни вправо/влево и нажми Сохранить ширину. После обновления страницы ширина должна остаться.