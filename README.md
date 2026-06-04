Отлично, editor.js вынесли правильно. Теперь добавляем секции страницы уже в нормальный отдельный JS-файл.

Сейчас делаем 3 правки:

1. editor.php — добавляем HTML панели секций и выбор секции у блока.
2. editor.js — добавляем логику секций.
3. editor.css — добавляем стили.


---

1. editor.php — панель секций

Файл:

/local/sitebuilder/editor.php

В правой колонке найди блок:

<div class="sb-panel">
    <h2 class="sb-panel-title">Свойства страницы</h2>

После закрытия этого блока </div> и перед блоком “Свойства блока” вставь:

<div class="sb-panel sb-page-sections-editor">
    <div class="sb-page-sections-editor__head">
        <div>
            <h2 class="sb-panel-title">Секции страницы</h2>
            <p class="sb-editor-note">
                Большие блоки страницы. Внутрь секций распределяются компоненты.
            </p>
        </div>

        <button class="sb-btn sb-btn-primary sb-btn-small" type="button" id="addPageSectionBtn">
            + Секция
        </button>
    </div>

    <div id="pageSectionsMessage" class="sb-page-sections-message" hidden></div>

    <div id="pageSectionsList" class="sb-page-sections-list">
        <div class="sb-empty">Выберите страницу</div>
    </div>
</div>


---

2. editor.php — выбор секции у блока

Найди в блоке “Свойства блока”:

<div class="sb-field">
    <label for="blockTypeInput">Тип</label>
    <input class="sb-input" type="text" id="blockTypeInput" disabled>
</div>

Сразу после него вставь:

<div class="sb-form-row sb-block-placement-row" style="margin-top:12px;">
    <div class="sb-field">
        <label for="blockSectionInput">Секция</label>
        <select class="sb-select" id="blockSectionInput">
            <option value="0">Основная секция</option>
        </select>
    </div>

    <div class="sb-field">
        <label for="blockColumnInput">Колонка</label>
        <select class="sb-select" id="blockColumnInput">
            <option value="1">Колонка 1</option>
        </select>
    </div>
</div>


---

3. editor.js — добавь поля в state

В твоём editor.js найди:

var state = {
    site: null,
    pages: [],
    currentPageId: 0,
    blocks: [],
    currentBlockId: 0,

Замени на:

var state = {
    site: null,
    pages: [],
    currentPageId: 0,
    blocks: [],
    currentBlockId: 0,
    pageSections: [],
    currentSectionId: 0,


---

4. editor.js — добавь функции секций

Вставь этот большой блок после функции buildPageTree() и перед async function loadSite():

/* =========================================================
   PAGE SECTIONS
   ========================================================= */

function apiData(res) {
    return res && res.data ? res.data : res;
}

function getCurrentPageId() {
    return Number(state.currentPageId || 0);
}

function getDefaultSectionId() {
    if (state.currentSectionId > 0) {
        return Number(state.currentSectionId);
    }

    if (state.pageSections.length) {
        return Number(state.pageSections[0].id || 0);
    }

    return 0;
}

function getSectionById(sectionId) {
    sectionId = Number(sectionId || 0);

    return state.pageSections.find(function (section) {
        return Number(section.id || 0) === sectionId;
    }) || null;
}

function getSectionColumns(sectionId) {
    var section = getSectionById(sectionId);

    if (!section) {
        return 1;
    }

    var layout = section.layout || {};
    var columns = Number(layout.columns || 1);

    if (columns < 1) columns = 1;
    if (columns > 4) columns = 4;

    return columns;
}

function setPageSectionsMessage(text, type) {
    var node = document.getElementById('pageSectionsMessage');

    if (!node) {
        return;
    }

    node.hidden = !text;
    node.textContent = text || '';
    node.className = 'sb-page-sections-message' + (type ? ' is-' + type : '');
}

async function loadPageSections() {
    var pageId = getCurrentPageId();
    var list = document.getElementById('pageSectionsList');

    if (!pageId) {
        state.pageSections = [];
        state.currentSectionId = 0;

        if (list) {
            list.innerHTML = '<div class="sb-empty">Выберите страницу</div>';
        }

        return;
    }

    try {
        var res = await api('pageSection.list', {
            siteId: siteId,
            pageId: pageId
        });

        var data = apiData(res);

        state.pageSections = Array.isArray(data.sections) ? data.sections : [];

        if (!state.currentSectionId && state.pageSections.length) {
            state.currentSectionId = Number(state.pageSections[0].id || 0);
        }

        if (
            state.currentSectionId &&
            !state.pageSections.some(function (section) {
                return Number(section.id || 0) === Number(state.currentSectionId);
            })
        ) {
            state.currentSectionId = state.pageSections.length ? Number(state.pageSections[0].id || 0) : 0;
        }

        renderPageSectionsPanel();
    } catch (e) {
        console.error(e);

        if (list) {
            list.innerHTML = '<div class="sb-empty">Не удалось загрузить секции</div>';
        }

        setPageSectionsMessage('Ошибка загрузки секций: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    }
}

function renderPageSectionsPanel() {
    var list = document.getElementById('pageSectionsList');

    if (!list) {
        return;
    }

    if (!state.currentPageId) {
        list.innerHTML = '<div class="sb-empty">Выберите страницу</div>';
        return;
    }

    if (!state.pageSections.length) {
        list.innerHTML = '<div class="sb-empty">Секций пока нет</div>';
        return;
    }

    list.innerHTML = state.pageSections.map(function (section, index) {
        var id = Number(section.id || 0);
        var title = section.title || 'Секция';
        var layout = section.layout || {};
        var props = section.props || {};
        var columns = Number(layout.columns || 1);
        var container = String(layout.container || 'default');
        var paddingTop = Number(props.paddingTop || 0);
        var paddingBottom = Number(props.paddingBottom || 0);
        var active = Number(state.currentSectionId || 0) === id ? ' is-active' : '';

        return ''
            + '<div class="sb-page-section-card' + active + '" data-page-section-id="' + id + '">'
            + '  <div class="sb-page-section-card__top" data-page-section-select="' + id + '">'
            + '      <div class="sb-page-section-card__index">' + (index + 1) + '</div>'
            + '      <div class="sb-page-section-card__main">'
            + '          <input class="sb-page-section-card__title-input" '
            + '                 type="text" '
            + '                 value="' + escapeHtml(title) + '" '
            + '                 data-section-field="title" '
            + '                 data-section-id="' + id + '">'
            + '          <div class="sb-page-section-card__meta">'
            + '              <span>' + columns + ' кол.</span>'
            + '              <span>' + escapeHtml(container) + '</span>'
            + '              <span>' + paddingTop + '/' + paddingBottom + 'px</span>'
            + '          </div>'
            + '      </div>'
            + '  </div>'
            + ''
            + '  <div class="sb-page-section-card__settings">'
            + '      <label>'
            + '          Колонки'
            + '          <select data-section-field="columns" data-section-id="' + id + '">'
            + '              <option value="1"' + (columns === 1 ? ' selected' : '') + '>1</option>'
            + '              <option value="2"' + (columns === 2 ? ' selected' : '') + '>2</option>'
            + '              <option value="3"' + (columns === 3 ? ' selected' : '') + '>3</option>'
            + '              <option value="4"' + (columns === 4 ? ' selected' : '') + '>4</option>'
            + '          </select>'
            + '      </label>'
            + ''
            + '      <label>'
            + '          Ширина'
            + '          <select data-section-field="container" data-section-id="' + id + '">'
            + '              <option value="default"' + (container === 'default' ? ' selected' : '') + '>Обычная</option>'
            + '              <option value="wide"' + (container === 'wide' ? ' selected' : '') + '>Широкая</option>'
            + '              <option value="full"' + (container === 'full' ? ' selected' : '') + '>На всю ширину</option>'
            + '          </select>'
            + '      </label>'
            + '  </div>'
            + ''
            + '  <div class="sb-page-section-card__actions">'
            + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-section-action="move-up" data-section-id="' + id + '">↑</button>'
            + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-section-action="move-down" data-section-id="' + id + '">↓</button>'
            + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-section-action="save" data-section-id="' + id + '">Сохранить</button>'
            + '      <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-section-action="delete" data-section-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }).join('');
}

function groupBlocksBySectionAndColumn() {
    var result = {};
    var firstSectionId = state.pageSections.length ? Number(state.pageSections[0].id || 0) : 0;

    state.pageSections.forEach(function (section) {
        var sectionId = Number(section.id || 0);
        var columns = getSectionColumns(sectionId);

        result[sectionId] = {};

        for (var i = 1; i <= columns; i++) {
            result[sectionId][i] = [];
        }
    });

    state.blocks.forEach(function (block) {
        var sectionId = Number(block.sectionId || 0);

        if (!sectionId || !result[sectionId]) {
            sectionId = firstSectionId;
        }

        if (!sectionId || !result[sectionId]) {
            return;
        }

        var columns = getSectionColumns(sectionId);
        var column = Number(block.column || 1);

        if (column < 1) column = 1;
        if (column > columns) column = columns;

        result[sectionId][column].push(block);
    });

    return result;
}

function fillBlockPlacementForm(block) {
    var sectionSelect = document.getElementById('blockSectionInput');
    var columnSelect = document.getElementById('blockColumnInput');

    if (!sectionSelect || !columnSelect) {
        return;
    }

    if (!block) {
        sectionSelect.innerHTML = '<option value="0">Нет секций</option>';
        columnSelect.innerHTML = '<option value="1">Колонка 1</option>';
        return;
    }

    if (!state.pageSections.length) {
        sectionSelect.innerHTML = '<option value="0">Нет секций</option>';
        columnSelect.innerHTML = '<option value="1">Колонка 1</option>';
        return;
    }

    var currentSectionId = Number(block.sectionId || 0);

    if (!currentSectionId || !getSectionById(currentSectionId)) {
        currentSectionId = getDefaultSectionId();
    }

    sectionSelect.innerHTML = state.pageSections.map(function (section) {
        var id = Number(section.id || 0);
        var title = section.title || ('Секция #' + id);

        return '<option value="' + id + '"' + (id === currentSectionId ? ' selected' : '') + '>' + escapeHtml(title) + '</option>';
    }).join('');

    var columns = getSectionColumns(currentSectionId);
    var currentColumn = Number(block.column || 1);

    if (currentColumn < 1) currentColumn = 1;
    if (currentColumn > columns) currentColumn = columns;

    var columnHtml = '';

    for (var i = 1; i <= columns; i++) {
        columnHtml += '<option value="' + i + '"' + (i === currentColumn ? ' selected' : '') + '>Колонка ' + i + '</option>';
    }

    columnSelect.innerHTML = columnHtml;
}

async function saveBlockPlacement(block) {
    if (!block) {
        return;
    }

    var sectionSelect = document.getElementById('blockSectionInput');
    var columnSelect = document.getElementById('blockColumnInput');

    if (!sectionSelect || !columnSelect) {
        return;
    }

    var sectionId = Number(sectionSelect.value || 0);
    var column = Number(columnSelect.value || 1);

    if (sectionId <= 0) {
        return;
    }

    await api('pageSection.assignBlock', {
        blockId: Number(block.id || 0),
        sectionId: sectionId,
        column: column
    });
}

async function assignBlockToSection(blockId, sectionId, column) {
    blockId = Number(blockId || 0);
    sectionId = Number(sectionId || 0);
    column = Number(column || 1);

    if (blockId <= 0 || sectionId <= 0) {
        return;
    }

    await api('pageSection.assignBlock', {
        blockId: blockId,
        sectionId: sectionId,
        column: column
    });
}

async function ensureUnsectionedBlocksAssigned() {
    var sectionId = getDefaultSectionId();

    if (!sectionId) {
        return;
    }

    var changed = false;

    for (var i = 0; i < state.blocks.length; i++) {
        var block = state.blocks[i];

        if (Number(block.sectionId || 0) > 0) {
            continue;
        }

        await assignBlockToSection(Number(block.id || 0), sectionId, 1);
        changed = true;
    }

    if (changed) {
        var res = await api('block.list', {
            pageId: state.currentPageId
        });

        state.blocks = Array.isArray(res.blocks) ? res.blocks : [];
    }
}

async function createPageSection() {
    var pageId = getCurrentPageId();

    if (!pageId) {
        alert('Сначала выберите страницу');
        return;
    }

    var title = prompt('Название секции', 'Новая секция');

    if (title === null) {
        return;
    }

    title = String(title || '').trim();

    if (!title) {
        title = 'Новая секция';
    }

    setPageSectionsMessage('Создаю секцию...', 'info');

    try {
        var res = await api('pageSection.create', {
            siteId: siteId,
            pageId: pageId,
            title: title,
            layout: JSON.stringify({
                container: 'default',
                columns: 1,
                gap: 24
            }),
            props: JSON.stringify({
                backgroundColor: '',
                backgroundImage: '',
                paddingTop: 40,
                paddingBottom: 40,
                minHeight: 0
            })
        });

        var data = apiData(res);

        state.pageSections = Array.isArray(data.sections) ? data.sections : [];
        state.currentSectionId = data.section && data.section.id ? Number(data.section.id) : getDefaultSectionId();

        renderPageSectionsPanel();
        renderBlocks();

        setPageSectionsMessage('Секция создана', 'success');
    } catch (e) {
        console.error(e);
        setPageSectionsMessage('Не удалось создать секцию: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    }
}

async function savePageSection(sectionId) {
    sectionId = Number(sectionId || 0);

    var section = getSectionById(sectionId);

    if (!section) {
        alert('Секция не найдена');
        return;
    }

    var card = document.querySelector('[data-page-section-id="' + sectionId + '"]');

    if (!card) {
        return;
    }

    var titleInput = card.querySelector('[data-section-field="title"]');
    var columnsSelect = card.querySelector('[data-section-field="columns"]');
    var containerSelect = card.querySelector('[data-section-field="container"]');

    var title = titleInput ? String(titleInput.value || '').trim() : section.title;
    var columns = columnsSelect ? Number(columnsSelect.value || 1) : Number((section.layout || {}).columns || 1);
    var container = containerSelect ? String(containerSelect.value || 'default') : String((section.layout || {}).container || 'default');

    var layout = Object.assign({}, section.layout || {}, {
        columns: columns,
        container: container
    });

    setPageSectionsMessage('Сохраняю секцию...', 'info');

    try {
        var res = await api('pageSection.update', {
            sectionId: sectionId,
            title: title,
            layout: JSON.stringify(layout)
        });

        var data = apiData(res);

        state.pageSections = Array.isArray(data.sections) ? data.sections : [];

        renderPageSectionsPanel();
        await loadBlocks();

        setPageSectionsMessage('Секция сохранена', 'success');
    } catch (e) {
        console.error(e);
        setPageSectionsMessage('Не удалось сохранить секцию: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    }
}

async function movePageSection(sectionId, dir) {
    sectionId = Number(sectionId || 0);

    if (!sectionId) {
        return;
    }

    try {
        var res = await api('pageSection.move', {
            sectionId: sectionId,
            dir: dir
        });

        var data = apiData(res);

        state.pageSections = Array.isArray(data.sections) ? data.sections : state.pageSections;

        renderPageSectionsPanel();
        renderBlocks();
    } catch (e) {
        console.error(e);
        setPageSectionsMessage('Не удалось переместить секцию: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    }
}

async function deletePageSection(sectionId) {
    sectionId = Number(sectionId || 0);

    if (!sectionId) {
        return;
    }

    var section = getSectionById(sectionId);
    var title = section && section.title ? section.title : 'секцию';

    if (!confirm('Удалить "' + title + '"? Компоненты будут перенесены в другую секцию.')) {
        return;
    }

    try {
        var res = await api('pageSection.delete', {
            sectionId: sectionId
        });

        var data = apiData(res);

        state.pageSections = Array.isArray(data.sections) ? data.sections : [];

        if (
            state.currentSectionId === sectionId ||
            !state.pageSections.some(function (s) {
                return Number(s.id || 0) === Number(state.currentSectionId);
            })
        ) {
            state.currentSectionId = state.pageSections.length ? Number(state.pageSections[0].id || 0) : 0;
        }

        renderPageSectionsPanel();
        await loadBlocks();

        setPageSectionsMessage('Секция удалена', 'success');
    } catch (e) {
        console.error(e);
        setPageSectionsMessage('Не удалось удалить секцию: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    }
}


---

5. editor.js — измени loadBlocks()

Найди функцию:

async function loadBlocks() {

Внутри неё найди этот кусок:

if (!state.currentPageId) {
    state.blocks = [];
    state.currentBlockId = 0;
    renderBlocks();
    fillBlockForm();
    return;
}

var res = await api('block.list', {
    pageId: state.currentPageId
});

Замени на:

if (!state.currentPageId) {
    state.blocks = [];
    state.currentBlockId = 0;
    state.pageSections = [];
    state.currentSectionId = 0;
    renderPageSectionsPanel();
    renderBlocks();
    fillBlockForm();
    return;
}

await loadPageSections();

var res = await api('block.list', {
    pageId: state.currentPageId
});

Ниже после:

state.blocks = Array.isArray(res.blocks) ? res.blocks : [];

добавь:

await ensureUnsectionedBlocksAssigned();


---

6. editor.js — замени renderBlocks()

Найди функцию:

function renderBlocks() {

Замени её полностью на:

function renderBlocks() {
    if (!state.currentPageId) {
        blocksList.innerHTML = ''
            + '<div class="sb-editor-empty-big">'
            + '   <strong>Страница не выбрана</strong>'
            + '   Выбери страницу слева, чтобы редактировать блоки'
            + '</div>';
        return;
    }

    if (!state.pageSections.length) {
        if (!state.blocks.length) {
            blocksList.innerHTML = ''
                + '<div class="sb-editor-empty-big">'
                + '   <strong>На странице пока нет блоков</strong>'
                + '   Добавь первый блок через панель сверху'
                + '</div>';
            return;
        }

        blocksList.innerHTML = state.blocks.map(function (block) {
            var active = Number(block.id || 0) === state.currentBlockId ? ' is-active' : '';

            return ''
                + '<div class="sb-editor-block' + active + '" data-block-id="' + Number(block.id || 0) + '">'
                + '  <div class="sb-editor-block-head">'
                + '      <div>'
                + '          <h3 class="sb-editor-block-title">' + escapeHtml(block.type || 'block') + '</h3>'
                + '          <div class="sb-editor-chip">block #' + Number(block.id || 0) + '</div>'
                + '      </div>'
                + '  </div>'
                + '  <div class="sb-editor-block-preview">' + escapeHtml(blockPreviewText(block)) + '</div>'
                + '</div>';
        }).join('');

        return;
    }

    var grouped = groupBlocksBySectionAndColumn();

    blocksList.innerHTML = state.pageSections.map(function (section) {
        var sectionId = Number(section.id || 0);
        var layout = section.layout || {};
        var columns = getSectionColumns(sectionId);
        var activeSection = Number(state.currentSectionId || 0) === sectionId ? ' is-active' : '';

        var html = ''
            + '<div class="sb-editor-section-preview' + activeSection + '" data-editor-section-id="' + sectionId + '">'
            + '  <div class="sb-editor-section-preview__head" data-page-section-select="' + sectionId + '">'
            + '      <div>'
            + '          <h3 class="sb-editor-section-preview__title">' + escapeHtml(section.title || 'Секция') + '</h3>'
            + '          <div class="sb-editor-section-preview__meta">'
            + '              <span>' + columns + ' кол.</span>'
            + '              <span>' + escapeHtml(layout.container || 'default') + '</span>'
            + '          </div>'
            + '      </div>'
            + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-add-block-to-section="' + sectionId + '">Выбрать</button>'
            + '  </div>'
            + '  <div class="sb-editor-section-preview__grid sb-editor-section-preview__grid--' + columns + '">';

        for (var column = 1; column <= columns; column++) {
            var blocks = grouped[sectionId] && grouped[sectionId][column]
                ? grouped[sectionId][column]
                : [];

            html += ''
                + '<div class="sb-editor-section-preview__column">'
                + '  <div class="sb-editor-section-preview__column-title">Колонка ' + column + '</div>';

            if (!blocks.length) {
                html += '<div class="sb-editor-section-preview__empty">Пусто</div>';
            } else {
                html += blocks.map(function (block) {
                    var active = Number(block.id || 0) === state.currentBlockId ? ' is-active' : '';

                    return ''
                        + '<div class="sb-editor-block' + active + '" data-block-id="' + Number(block.id || 0) + '">'
                        + '  <div class="sb-editor-block-head">'
                        + '      <div>'
                        + '          <h3 class="sb-editor-block-title">' + escapeHtml(block.type || 'block') + '</h3>'
                        + '          <div class="sb-editor-chip">block #' + Number(block.id || 0) + '</div>'
                        + '      </div>'
                        + '  </div>'
                        + '  <div class="sb-editor-block-preview">' + escapeHtml(blockPreviewText(block)) + '</div>'
                        + '</div>';
                }).join('');
            }

            html += '</div>';
        }

        html += ''
            + '  </div>'
            + '</div>';

        return html;
    }).join('');
}


---

7. editor.js — доработай fillBlockForm()

Найди внутри fillBlockForm() блок:

if (!block) {

Перед return; внутри этого блока добавь:

fillBlockPlacementForm(null);

Дальше ниже в конце функции, после:

fillVisualBlockForm(block);

добавь:

fillBlockPlacementForm(block);


---

8. editor.js — измени createBlock(type)

В функции createBlock(type) перед:

await api('block.create', {

добавь:

var targetSectionId = getDefaultSectionId();

Затем замени:

await api('block.create', {
    pageId: state.currentPageId,
    type: type,
    content: JSON.stringify(content),
    props: JSON.stringify(props)
});

await loadBlocks();

на:

var createRes = await api('block.create', {
    pageId: state.currentPageId,
    type: type,
    content: JSON.stringify(content),
    props: JSON.stringify(props),
    sectionId: targetSectionId,
    column: 1
});

await loadBlocks();

var createdBlockId = Number(
    (createRes.block && createRes.block.id) ||
    (createRes.data && createRes.data.block && createRes.data.block.id) ||
    0
);

if (!createdBlockId && state.blocks.length) {
    var sortedBlocks = state.blocks.slice().sort(function (a, b) {
        return Number(b.id || 0) - Number(a.id || 0);
    });

    createdBlockId = Number(sortedBlocks[0].id || 0);
}

if (createdBlockId > 0 && targetSectionId > 0) {
    await assignBlockToSection(createdBlockId, targetSectionId, 1);
    await loadBlocks();
}


---

9. editor.js — измени saveBlock()

Найди:

await api('block.update', {
    id: block.id,
    content: JSON.stringify(collected.content),
    props: JSON.stringify(collected.props)
});

await loadBlocks();

Замени на:

await api('block.update', {
    id: block.id,
    content: JSON.stringify(collected.content),
    props: JSON.stringify(collected.props)
});

await saveBlockPlacement(block);

await loadBlocks();


---

10. editor.js — после выбора страницы загружай секции

Найди:

pagesList.addEventListener('click', async function (e) {

Внутри в конце сейчас:

await loadBlocks();

Этого достаточно, потому что loadBlocks() теперь сам грузит секции. Ничего менять не надо.


---

11. editor.js — добавь обработчики секций

Перед:

document.getElementById('createPageBtn').addEventListener('click', createPage);

вставь:

var addPageSectionBtn = document.getElementById('addPageSectionBtn');
if (addPageSectionBtn) {
    addPageSectionBtn.addEventListener('click', createPageSection);
}

document.addEventListener('click', function (e) {
    var selectSection = e.target.closest('[data-page-section-select], [data-add-block-to-section]');

    if (selectSection) {
        var sectionId = Number(
            selectSection.getAttribute('data-page-section-select') ||
            selectSection.getAttribute('data-add-block-to-section') ||
            0
        );

        if (sectionId > 0) {
            state.currentSectionId = sectionId;
            renderPageSectionsPanel();
            renderBlocks();
        }

        return;
    }

    var sectionBtn = e.target.closest('[data-section-action]');

    if (!sectionBtn) {
        return;
    }

    var action = sectionBtn.getAttribute('data-section-action');
    var sectionId = Number(sectionBtn.getAttribute('data-section-id') || 0);

    if (action === 'move-up') {
        movePageSection(sectionId, 'up');
        return;
    }

    if (action === 'move-down') {
        movePageSection(sectionId, 'down');
        return;
    }

    if (action === 'save') {
        savePageSection(sectionId);
        return;
    }

    if (action === 'delete') {
        deletePageSection(sectionId);
    }
});

document.addEventListener('change', function (e) {
    var sectionField = e.target.closest('[data-section-field="columns"], [data-section-field="container"]');

    if (sectionField) {
        var sectionId = Number(sectionField.getAttribute('data-section-id') || 0);

        if (sectionId > 0) {
            savePageSection(sectionId);
        }

        return;
    }

    if (e.target && e.target.id === 'blockSectionInput') {
        var block = getCurrentBlock();

        if (block) {
            fillBlockPlacementForm(Object.assign({}, block, {
                sectionId: Number(e.target.value || 0),
                column: 1
            }));
        }
    }
});


---

12. editor.css — добавь стили

Файл:

/local/sitebuilder/assets/admin/editor.css

В конец добавь:

/* =========================================================
   PAGE SECTIONS EDITOR
   ========================================================= */

.sb-page-sections-editor {
    margin-top: 14px;
}

.sb-page-sections-editor__head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
}

.sb-page-sections-message {
    margin: 10px 0;
    padding: 9px 10px;
    border-radius: 12px;
    background: #f3f4f6;
    color: #374151;
    font-size: 12px;
    line-height: 1.35;
}

.sb-page-sections-message.is-success {
    background: #dcfce7;
    color: #166534;
}

.sb-page-sections-message.is-error {
    background: #fee2e2;
    color: #991b1b;
}

.sb-page-sections-message.is-info {
    background: #eef2ff;
    color: #3730a3;
}

.sb-page-sections-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
    margin-top: 12px;
}

.sb-page-section-card {
    padding: 12px;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    background: #f9fafb;
}

.sb-page-section-card.is-active {
    border-color: #2563eb;
    background: #eff6ff;
}

.sb-page-section-card__top {
    display: flex;
    align-items: flex-start;
    gap: 10px;
    cursor: pointer;
}

.sb-page-section-card__index {
    width: 28px;
    height: 28px;
    flex: 0 0 28px;
    border-radius: 10px;
    background: #eef2ff;
    color: #3730a3;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 12px;
    font-weight: 900;
}

.sb-page-section-card__main {
    min-width: 0;
    flex: 1;
}

.sb-page-section-card__title-input {
    width: 100%;
    height: 32px;
    padding: 0 10px;
    border: 1px solid #dbe3ef;
    border-radius: 10px;
    background: #fff;
    color: #111827;
    font-size: 13px;
    font-weight: 800;
    outline: none;
}

.sb-page-section-card__title-input:focus {
    border-color: #2563eb;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .12);
}

.sb-page-section-card__meta {
    display: flex;
    flex-wrap: wrap;
    gap: 5px;
    margin-top: 7px;
}

.sb-page-section-card__meta span {
    display: inline-flex;
    align-items: center;
    min-height: 22px;
    padding: 0 7px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #6b7280;
    font-size: 11px;
    font-weight: 700;
}

.sb-page-section-card__settings {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 8px;
    margin-top: 10px;
}

.sb-page-section-card__settings label {
    display: flex;
    flex-direction: column;
    gap: 5px;
    color: #374151;
    font-size: 11px;
    font-weight: 800;
}

.sb-page-section-card__settings select {
    width: 100%;
    height: 32px;
    padding: 0 9px;
    border: 1px solid #dbe3ef;
    border-radius: 10px;
    background: #fff;
    color: #111827;
    font-size: 12px;
    outline: none;
}

.sb-page-section-card__actions {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 10px;
}

.sb-block-placement-row {
    display: grid;
    grid-template-columns: 1fr 120px;
    gap: 10px;
}

.sb-editor-section-preview {
    margin-bottom: 16px;
    padding: 14px;
    border: 1px solid #e5e7eb;
    border-radius: 18px;
    background: #ffffff;
}

.sb-editor-section-preview.is-active {
    border-color: #2563eb;
    box-shadow: 0 0 0 3px rgba(37, 99, 235, .08);
}

.sb-editor-section-preview__head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 12px;
    cursor: pointer;
}

.sb-editor-section-preview__title {
    margin: 0;
    color: #111827;
    font-size: 15px;
    font-weight: 900;
}

.sb-editor-section-preview__meta {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 6px;
}

.sb-editor-section-preview__meta span {
    display: inline-flex;
    min-height: 22px;
    align-items: center;
    padding: 0 8px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #6b7280;
    font-size: 11px;
    font-weight: 700;
}

.sb-editor-section-preview__grid {
    display: grid;
    gap: 10px;
}

.sb-editor-section-preview__grid--1 {
    grid-template-columns: 1fr;
}

.sb-editor-section-preview__grid--2 {
    grid-template-columns: repeat(2, minmax(0, 1fr));
}

.sb-editor-section-preview__grid--3 {
    grid-template-columns: repeat(3, minmax(0, 1fr));
}

.sb-editor-section-preview__grid--4 {
    grid-template-columns: repeat(4, minmax(0, 1fr));
}

.sb-editor-section-preview__column {
    min-width: 0;
    padding: 10px;
    border: 1px dashed #cbd5e1;
    border-radius: 14px;
    background: #f8fafc;
}

.sb-editor-section-preview__column-title {
    margin-bottom: 8px;
    color: #64748b;
    font-size: 11px;
    font-weight: 900;
    text-transform: uppercase;
    letter-spacing: .03em;
}

.sb-editor-section-preview__empty {
    padding: 12px;
    border-radius: 12px;
    background: #fff;
    color: #9ca3af;
    font-size: 12px;
    text-align: center;
}

@media (max-width: 1200px) {
    .sb-editor-section-preview__grid--2,
    .sb-editor-section-preview__grid--3,
    .sb-editor-section-preview__grid--4 {
        grid-template-columns: 1fr;
    }

    .sb-block-placement-row {
        grid-template-columns: 1fr;
    }
}


---

13. Обнови подключение editor.js

В editor.php поставь версию:

<script src="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.js?v=2"></script>


---

После этого проверь:

1. Открывается редактор.
2. Справа появилась панель “Секции страницы”.
3. Есть “Основная секция”.
4. Можно нажать “+ Секция”.
5. Можно выбрать 2 колонки.
6. В свойствах блока появился выбор секции и колонки.
7. На публичной странице блоки выводятся внутри секций.

Если после сохранения секции появится UNKNOWN_ACTION, значит в /local/sitebuilder/api/index.php ещё не подключён обработчик pageSection.*. Тогда пришли мне api/index.php, и я дам готовый вариант.