Идём дальше: добавляем в editor.php панель “Секции страницы”.

Пока делаем базовый рабочий вариант:

1. Видим секции текущей страницы.
2. Можем добавить секцию.
3. Можем двигать секцию вверх/вниз.
4. Можем удалить секцию.
5. Можем менять количество колонок 1/2/3/4.

Компоненты внутрь секций начнём распределять следующим шагом.


---

1. В editor.php добавь HTML-блок секций

Файл:

/local/sitebuilder/editor.php

Найди место в правой панели, где у тебя свойства страницы/блоков. Туда вставь этот блок:

<div class="sb-editor-card sb-page-sections-editor">
    <div class="sb-page-sections-editor__head">
        <div>
            <div class="sb-page-sections-editor__title">Секции страницы</div>
            <div class="sb-page-sections-editor__subtitle">
                Большие блоки страницы, как в Tilda
            </div>
        </div>

        <button class="sb-btn sb-btn-primary sb-btn-small" type="button" id="addPageSectionBtn">
            + Секция
        </button>
    </div>

    <div id="pageSectionsMessage" class="sb-page-sections-message" hidden></div>

    <div id="pageSectionsList" class="sb-page-sections-list">
        <div class="sb-page-sections-empty">Выберите страницу</div>
    </div>
</div>

Если класса sb-editor-card у тебя нет, ничего страшного — CSS ниже всё подправит.


---

2. В editor.php добавь JS

Внутри основного <script> внизу файла добавь этот код.

Лучше вставить ближе к концу, но до закрывающего </script>:

/* =========================================================
   PAGE SECTIONS EDITOR
   ========================================================= */

var pageSectionsState = {
    pageId: 0,
    sections: [],
    loading: false
};

function sbGetCurrentEditorPageId() {
    if (window.state && state.currentPage && state.currentPage.id) {
        return Number(state.currentPage.id || 0);
    }

    if (window.state && state.page && state.page.id) {
        return Number(state.page.id || 0);
    }

    if (window.currentPage && currentPage.id) {
        return Number(currentPage.id || 0);
    }

    if (typeof currentPageId !== 'undefined' && currentPageId) {
        return Number(currentPageId || 0);
    }

    var activePage =
        document.querySelector('[data-page-id].is-active') ||
        document.querySelector('[data-page-id].active') ||
        document.querySelector('.is-active[data-id][data-type="page"]') ||
        document.querySelector('[data-role="page-item"].is-active');

    if (activePage) {
        return Number(
            activePage.getAttribute('data-page-id') ||
            activePage.getAttribute('data-id') ||
            0
        );
    }

    var pageInput =
        document.querySelector('[name="pageId"]') ||
        document.querySelector('#pageId') ||
        document.querySelector('[data-current-page-id]');

    if (pageInput) {
        return Number(
            pageInput.value ||
            pageInput.getAttribute('data-current-page-id') ||
            0
        );
    }

    var params = new URLSearchParams(window.location.search);
    return Number(params.get('pageId') || 0);
}

function sbPageSectionsSetMessage(text, type) {
    var node = document.getElementById('pageSectionsMessage');

    if (!node) {
        return;
    }

    node.hidden = !text;
    node.textContent = text || '';
    node.className = 'sb-page-sections-message' + (type ? ' is-' + type : '');
}

function sbPageSectionsEscape(value) {
    return String(value == null ? '' : value)
        .replace(/&/g, '&amp;')
        .replace(/</g, '&lt;')
        .replace(/>/g, '&gt;')
        .replace(/"/g, '&quot;')
        .replace(/'/g, '&#039;');
}

async function sbLoadPageSections(forcePageId) {
    var pageId = Number(forcePageId || sbGetCurrentEditorPageId() || 0);
    var list = document.getElementById('pageSectionsList');

    if (!list) {
        return;
    }

    pageSectionsState.pageId = pageId;

    if (!pageId) {
        pageSectionsState.sections = [];
        list.innerHTML = '<div class="sb-page-sections-empty">Выберите страницу</div>';
        return;
    }

    pageSectionsState.loading = true;
    list.innerHTML = '<div class="sb-page-sections-empty">Загрузка секций...</div>';
    sbPageSectionsSetMessage('', '');

    try {
        var data = await api('pageSection.list', {
            siteId: siteId,
            pageId: pageId
        });

        pageSectionsState.sections = Array.isArray(data.sections) ? data.sections : [];
        sbRenderPageSections();
    } catch (e) {
        console.error(e);
        list.innerHTML = '<div class="sb-page-sections-empty">Не удалось загрузить секции</div>';
        sbPageSectionsSetMessage('Ошибка загрузки секций: ' + (e.message || e), 'error');
    } finally {
        pageSectionsState.loading = false;
    }
}

function sbRenderPageSections() {
    var list = document.getElementById('pageSectionsList');

    if (!list) {
        return;
    }

    var sections = pageSectionsState.sections || [];

    if (!sections.length) {
        list.innerHTML = '<div class="sb-page-sections-empty">Секций пока нет</div>';
        return;
    }

    list.innerHTML = sections.map(function (section, index) {
        var id = Number(section.id || 0);
        var title = section.title || 'Секция';
        var layout = section.layout || {};
        var props = section.props || {};
        var columns = Number(layout.columns || 1);
        var container = String(layout.container || 'default');
        var paddingTop = Number(props.paddingTop || 0);
        var paddingBottom = Number(props.paddingBottom || 0);

        return ''
            + '<div class="sb-page-section-card" data-page-section-id="' + id + '">'
            + '  <div class="sb-page-section-card__top">'
            + '      <div class="sb-page-section-card__index">' + (index + 1) + '</div>'
            + '      <div class="sb-page-section-card__main">'
            + '          <input class="sb-page-section-card__title-input" '
            + '                 type="text" '
            + '                 value="' + sbPageSectionsEscape(title) + '" '
            + '                 data-section-field="title" '
            + '                 data-section-id="' + id + '">'
            + '          <div class="sb-page-section-card__meta">'
            + '              <span>' + columns + ' кол.</span>'
            + '              <span>' + sbPageSectionsEscape(container) + '</span>'
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

function sbFindPageSection(sectionId) {
    sectionId = Number(sectionId || 0);

    for (var i = 0; i < pageSectionsState.sections.length; i++) {
        if (Number(pageSectionsState.sections[i].id || 0) === sectionId) {
            return pageSectionsState.sections[i];
        }
    }

    return null;
}

async function sbCreatePageSection() {
    var pageId = Number(pageSectionsState.pageId || sbGetCurrentEditorPageId() || 0);

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

    sbPageSectionsSetMessage('Создаю секцию...', 'info');

    try {
        var data = await api('pageSection.create', {
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

        pageSectionsState.sections = Array.isArray(data.sections) ? data.sections : [];
        sbRenderPageSections();
        sbPageSectionsSetMessage('Секция создана', 'success');
    } catch (e) {
        console.error(e);
        sbPageSectionsSetMessage('Не удалось создать секцию: ' + (e.message || e), 'error');
    }
}

async function sbSavePageSection(sectionId) {
    sectionId = Number(sectionId || 0);

    var section = sbFindPageSection(sectionId);

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

    sbPageSectionsSetMessage('Сохраняю секцию...', 'info');

    try {
        var data = await api('pageSection.update', {
            sectionId: sectionId,
            title: title,
            layout: JSON.stringify(layout)
        });

        pageSectionsState.sections = Array.isArray(data.sections) ? data.sections : [];
        sbRenderPageSections();
        sbPageSectionsSetMessage('Секция сохранена', 'success');
    } catch (e) {
        console.error(e);
        sbPageSectionsSetMessage('Не удалось сохранить секцию: ' + (e.message || e), 'error');
    }
}

async function sbMovePageSection(sectionId, dir) {
    sectionId = Number(sectionId || 0);

    if (!sectionId) {
        return;
    }

    try {
        var data = await api('pageSection.move', {
            sectionId: sectionId,
            dir: dir
        });

        pageSectionsState.sections = Array.isArray(data.sections) ? data.sections : pageSectionsState.sections;
        sbRenderPageSections();
    } catch (e) {
        console.error(e);
        sbPageSectionsSetMessage('Не удалось переместить секцию: ' + (e.message || e), 'error');
    }
}

async function sbDeletePageSection(sectionId) {
    sectionId = Number(sectionId || 0);

    if (!sectionId) {
        return;
    }

    var section = sbFindPageSection(sectionId);
    var title = section && section.title ? section.title : 'секцию';

    if (!confirm('Удалить "' + title + '"? Компоненты будут перенесены в другую секцию.')) {
        return;
    }

    try {
        var data = await api('pageSection.delete', {
            sectionId: sectionId
        });

        pageSectionsState.sections = Array.isArray(data.sections) ? data.sections : [];
        sbRenderPageSections();
        sbPageSectionsSetMessage('Секция удалена', 'success');
    } catch (e) {
        console.error(e);
        sbPageSectionsSetMessage('Не удалось удалить секцию: ' + (e.message || e), 'error');
    }
}

function sbBindPageSectionsEditor() {
    var addBtn = document.getElementById('addPageSectionBtn');

    if (addBtn && !addBtn.dataset.bound) {
        addBtn.dataset.bound = '1';
        addBtn.addEventListener('click', sbCreatePageSection);
    }

    document.addEventListener('click', function (e) {
        var btn = e.target.closest('[data-section-action]');

        if (!btn) {
            return;
        }

        var action = btn.getAttribute('data-section-action');
        var sectionId = Number(btn.getAttribute('data-section-id') || 0);

        if (action === 'move-up') {
            sbMovePageSection(sectionId, 'up');
            return;
        }

        if (action === 'move-down') {
            sbMovePageSection(sectionId, 'down');
            return;
        }

        if (action === 'save') {
            sbSavePageSection(sectionId);
            return;
        }

        if (action === 'delete') {
            sbDeletePageSection(sectionId);
        }
    });

    document.addEventListener('change', function (e) {
        var field = e.target.closest('[data-section-field="columns"], [data-section-field="container"]');

        if (!field) {
            return;
        }

        var sectionId = Number(field.getAttribute('data-section-id') || 0);

        if (sectionId > 0) {
            sbSavePageSection(sectionId);
        }
    });
}

function sbInitPageSectionsEditor() {
    sbBindPageSectionsEditor();
    sbLoadPageSections();
}

document.addEventListener('DOMContentLoaded', function () {
    sbInitPageSectionsEditor();
});

/*
 * Если у тебя при выборе страницы уже есть своя функция loadPage/loadBlocks,
 * после выбора страницы можно вызвать:
 *
 * sbLoadPageSections(НОВЫЙ_PAGE_ID);
 */


---

3. Подключи секции после выбора страницы

Теперь важный момент.

Когда ты в редакторе кликаешь другую страницу, старые блоки наверняка перезагружаются через какую-то функцию. После этой функции нужно вызвать:

sbLoadPageSections(pageId);

Если у тебя в коде есть функция типа:

selectPage(pageId)
loadPage(pageId)
openPage(pageId)
setCurrentPage(page)

в конце неё добавь:

sbLoadPageSections(pageId);

Если не найдёшь — пока не страшно, при первом открытии редактора секции текущей страницы загрузятся. Потом мы подцепим к выбору страниц точнее.


---

4. Добавь CSS в editor.css

Файл:

/local/sitebuilder/assets/admin/editor.css

В конец добавь:

/* =========================================================
   PAGE SECTIONS EDITOR
   ========================================================= */

.sb-page-sections-editor {
    margin-top: 14px;
    padding: 14px;
    border: 1px solid #e5e7eb;
    border-radius: 18px;
    background: #ffffff;
}

.sb-page-sections-editor__head {
    display: flex;
    align-items: flex-start;
    justify-content: space-between;
    gap: 12px;
    margin-bottom: 12px;
}

.sb-page-sections-editor__title {
    color: #111827;
    font-size: 15px;
    font-weight: 900;
    line-height: 1.25;
}

.sb-page-sections-editor__subtitle {
    margin-top: 3px;
    color: #6b7280;
    font-size: 12px;
    line-height: 1.35;
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
}

.sb-page-sections-empty {
    padding: 16px;
    border: 1px dashed #cbd5e1;
    border-radius: 14px;
    background: #f8fafc;
    color: #6b7280;
    font-size: 13px;
    text-align: center;
}

.sb-page-section-card {
    padding: 12px;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
    background: #f9fafb;
}

.sb-page-section-card:hover {
    border-color: #c7d2fe;
    background: #f8fbff;
}

.sb-page-section-card__top {
    display: flex;
    align-items: flex-start;
    gap: 10px;
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

.sb-page-section-card__actions .sb-btn {
    min-height: 30px;
}


---

После этого в редакторе появится базовая панель секций.

Следующий шаг — сделать, чтобы компоненты/старые блоки можно было назначать в секцию и колонку, то есть прямо:

Компонент “Диск”
→ Секция “Документы”
→ Колонка 1

И тогда структура будет уже совсем похожа на Tilda.