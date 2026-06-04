Идём дальше: сделаем, чтобы компоненты добавлялись не просто на страницу, а прямо в выбранную секцию и колонку.

Сейчас получится так:

1. В каждой колонке появится кнопка “Добавлять сюда”.
2. Нажимаешь её.
3. Потом сверху выбираешь “Заголовок / Текст / Кнопка / HTML / Диск”.
4. Новый компонент создаётся именно в выбранной секции и колонке.


---

1. editor.js — добавь currentColumn

Найди в state:

pageSections: [],
currentSectionId: 0,

Замени на:

pageSections: [],
currentSectionId: 0,
currentColumn: 1,


---

2. editor.js — добавь функцию получения колонки

Найди функцию:

function getDefaultSectionId() {

После неё добавь:

function getDefaultColumn() {
    var sectionId = getDefaultSectionId();
    var columns = getSectionColumns(sectionId);
    var column = Number(state.currentColumn || 1);

    if (column < 1) column = 1;
    if (column > columns) column = columns;

    return column;
}


---

3. editor.js — при смене страницы сбрасывай колонку

Найди обработчик:

pagesList.addEventListener('click', async function (e) {

Внутри него есть:

state.currentPageId = Number(item.getAttribute('data-page-id') || 0);
state.currentBlockId = 0;

Сразу после добавь:

state.currentSectionId = 0;
state.currentColumn = 1;


---

4. editor.js — измени выбор секции

Найди в обработчиках секций этот кусок:

if (sectionId > 0) {
    state.currentSectionId = sectionId;
    renderPageSectionsPanel();
    renderBlocks();
}

Замени на:

if (sectionId > 0) {
    state.currentSectionId = sectionId;
    state.currentColumn = 1;
    renderPageSectionsPanel();
    renderBlocks();
}


---

5. editor.js — замени часть renderBlocks()

Внутри renderBlocks() найди этот кусок внутри цикла колонок:

html += ''
    + '<div class="sb-editor-section-preview__column">'
    + '  <div class="sb-editor-section-preview__column-title">Колонка ' + column + '</div>';

Замени на:

var isTargetColumn =
    Number(state.currentSectionId || 0) === sectionId &&
    Number(state.currentColumn || 1) === column;

html += ''
    + '<div class="sb-editor-section-preview__column' + (isTargetColumn ? ' is-target' : '') + '" data-section-id="' + sectionId + '" data-column="' + column + '">'
    + '  <div class="sb-editor-section-preview__column-head">'
    + '      <div class="sb-editor-section-preview__column-title">Колонка ' + column + '</div>'
    + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-set-add-target="' + sectionId + '" data-column="' + column + '">'
    +          (isTargetColumn ? 'Выбрано' : 'Добавлять сюда')
    + '      </button>'
    + '  </div>';


---

6. editor.js — измени createBlock(type)

Найди в createBlock(type):

var targetSectionId = getDefaultSectionId();

Сразу после добавь:

var targetColumn = getDefaultColumn();

Ниже найди:

sectionId: targetSectionId,
column: 1

Замени на:

sectionId: targetSectionId,
column: targetColumn

Ещё ниже найди:

await assignBlockToSection(createdBlockId, targetSectionId, 1);

Замени на:

await assignBlockToSection(createdBlockId, targetSectionId, targetColumn);


---

7. editor.js — добавь обработчик кнопки “Добавлять сюда”

В общем document.addEventListener('click', function (e) { ... }), который мы добавляли для секций, найди начало:

document.addEventListener('click', function (e) {
    var selectSection = e.target.closest('[data-page-section-select], [data-add-block-to-section]');

Перед строкой:

var selectSection = e.target.closest

добавь:

var addTargetBtn = e.target.closest('[data-set-add-target]');
if (addTargetBtn) {
    var targetSectionId = Number(addTargetBtn.getAttribute('data-set-add-target') || 0);
    var targetColumn = Number(addTargetBtn.getAttribute('data-column') || 1);

    if (targetSectionId > 0) {
        state.currentSectionId = targetSectionId;
        state.currentColumn = targetColumn > 0 ? targetColumn : 1;

        renderPageSectionsPanel();
        renderBlocks();

        setPageSectionsMessage(
            'Новые компоненты будут добавляться в секцию #' + targetSectionId + ', колонку ' + state.currentColumn,
            'success'
        );
    }

    return;
}


---

8. editor.js — при выборе блока запоминай его секцию и колонку

Найди:

blocksList.addEventListener('click', function (e) {

Внутри после:

state.currentBlockId = Number(item.getAttribute('data-block-id') || 0);

добавь:

var selectedBlock = getCurrentBlock();

if (selectedBlock) {
    var selectedSectionId = Number(selectedBlock.sectionId || 0);
    var selectedColumn = Number(selectedBlock.column || 1);

    if (selectedSectionId > 0) {
        state.currentSectionId = selectedSectionId;
    }

    state.currentColumn = selectedColumn > 0 ? selectedColumn : 1;
}


---

9. editor.js — при смене секции у блока сразу меняй колонки

У тебя уже есть обработчик:

if (e.target && e.target.id === 'blockSectionInput') {

Замени его целиком на:

if (e.target && e.target.id === 'blockSectionInput') {
    var block = getCurrentBlock();
    var newSectionId = Number(e.target.value || 0);

    if (block) {
        state.currentSectionId = newSectionId;
        state.currentColumn = 1;

        fillBlockPlacementForm(Object.assign({}, block, {
            sectionId: newSectionId,
            column: 1
        }));

        renderPageSectionsPanel();
        renderBlocks();
    }

    return;
}

if (e.target && e.target.id === 'blockColumnInput') {
    var columnValue = Number(e.target.value || 1);

    state.currentColumn = columnValue > 0 ? columnValue : 1;

    renderBlocks();
    return;
}


---

10. editor.css — добавь стили выбранной колонки

В конец файла:

/* =========================================================
   SECTION COLUMN TARGET
   ========================================================= */

.sb-editor-section-preview__column.is-target {
    border-color: #2563eb;
    background: #eff6ff;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .08);
}

.sb-editor-section-preview__column-head {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 10px;
    margin-bottom: 8px;
}

.sb-editor-section-preview__column-head .sb-btn {
    min-height: 26px;
    padding: 0 8px;
    font-size: 11px;
}


---

11. Обнови версию editor.js

В editor.php:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.js?v=3"></script>


---

После этого проверь:

1. Создай секцию с 2 колонками.
2. Нажми “Добавлять сюда” во второй колонке.
3. Добавь блок “Текст”.
4. Он должен появиться во второй колонке.
5. Выбери блок.
6. Справа в свойствах блока должна быть его секция и колонка.

Следующим шагом можно сделать уже более удобно: перетаскивание компонента мышкой между колонками, как в конструкторе.