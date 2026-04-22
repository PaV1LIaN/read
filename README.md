<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

global $APPLICATION, $USER;

if (!$USER->IsAuthorized()) {
    require $_SERVER['DOCUMENT_ROOT'] . '/auth.php';
    exit;
}

$siteId = (int)($_GET['siteId'] ?? 0);
if ($siteId <= 0) {
    die('SITE_ID_REQUIRED');
}

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>Редактор сайта</title>
    <?php $APPLICATION->ShowHead(); ?>
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    <style>
        .sb-editor-shell {
            display: grid;
            grid-template-columns: 340px minmax(0, 1fr) 380px;
            gap: 20px;
            align-items: start;
        }

        .sb-editor-column {
            min-width: 0;
        }

        .sb-editor-sticky {
            position: sticky;
            top: 16px;
        }

        .sb-editor-preview {
            min-height: 640px;
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 16px;
            box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
            overflow: hidden;
        }

        .sb-editor-preview__head {
            padding: 14px 18px;
            border-bottom: 1px solid #e5e7eb;
            display: flex;
            justify-content: space-between;
            align-items: center;
            background: #f9fafb;
        }

        .sb-editor-preview__body {
            padding: 20px;
            background: #f8fafc;
            min-height: 560px;
        }

        .sb-editor-page-preview {
            background: #fff;
            border: 1px dashed #d1d5db;
            border-radius: 16px;
            padding: 20px;
            min-height: 520px;
        }

        .sb-editor-page-title {
            margin: 0 0 18px;
            font-size: 28px;
            line-height: 1.2;
            font-weight: 700;
            color: #111827;
        }

        .sb-editor-block {
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            background: #fff;
            padding: 14px;
            margin-bottom: 14px;
            transition: border-color .15s ease, box-shadow .15s ease, background .15s ease;
            cursor: pointer;
        }

        .sb-editor-block:hover {
            border-color: #bfdbfe;
            box-shadow: 0 4px 14px rgba(37, 99, 235, 0.08);
        }

        .sb-editor-block.is-active {
            border-color: #2563eb;
            background: #eff6ff;
            box-shadow: 0 6px 18px rgba(37, 99, 235, 0.12);
        }

        .sb-editor-block__meta {
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 10px;
            margin-bottom: 10px;
        }

        .sb-editor-block__type {
            display: inline-flex;
            align-items: center;
            gap: 6px;
            background: #eef2ff;
            color: #1e3a8a;
            border-radius: 999px;
            padding: 4px 10px;
            font-size: 12px;
            font-weight: 700;
        }

        .sb-editor-block__actions {
            display: flex;
            gap: 8px;
            flex-wrap: wrap;
        }

        .sb-editor-block__content {
            color: #374151;
            font-size: 14px;
            line-height: 1.6;
        }

        .sb-editor-tree {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .sb-editor-page-item {
            border: 1px solid #e5e7eb;
            background: #fafafa;
            border-radius: 12px;
            padding: 12px;
            cursor: pointer;
            transition: border-color .15s ease, background .15s ease;
        }

        .sb-editor-page-item:hover {
            border-color: #bfdbfe;
            background: #fff;
        }

        .sb-editor-page-item.is-active {
            border-color: #2563eb;
            background: #eff6ff;
        }

        .sb-editor-page-item__title {
            margin: 0 0 6px;
            font-size: 15px;
            font-weight: 700;
            color: #111827;
        }

        .sb-editor-page-item__meta {
            font-size: 12px;
            color: #6b7280;
        }

        .sb-editor-empty {
            padding: 18px;
            border: 1px dashed #d1d5db;
            border-radius: 12px;
            color: #6b7280;
            background: #fff;
            text-align: center;
        }

        .sb-editor-form-grid {
            display: grid;
            grid-template-columns: 1fr;
            gap: 12px;
        }

        .sb-editor-toolbar {
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
            align-items: center;
        }

        .sb-editor-inspector-actions {
            display: flex;
            gap: 8px;
            flex-wrap: wrap;
        }

        .sb-editor-section-title {
            margin: 0 0 12px;
            font-size: 16px;
            font-weight: 700;
        }

        .sb-editor-divider {
            height: 1px;
            background: #e5e7eb;
            margin: 16px 0;
        }

        @media (max-width: 1280px) {
            .sb-editor-shell {
                grid-template-columns: 320px minmax(0, 1fr);
            }

            .sb-editor-column--right {
                grid-column: 1 / -1;
            }
        }

        @media (max-width: 900px) {
            .sb-editor-shell {
                grid-template-columns: 1fr;
            }

            .sb-editor-sticky {
                position: static;
            }
        }
    </style>
</head>
<body class="sb-admin-body">
<div class="sb-page">
    <div class="sb-topbar">
        <div class="sb-topbar-left">
            <a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← Назад к сайтам</a>
            <h1 id="siteTitle">Редактор сайта</h1>
            <p class="sb-subtitle" id="siteSubtitle">Загрузка данных...</p>
        </div>
        <div class="sb-userbox">
            Пользователь: <?= htmlspecialchars((string)$USER->GetLogin(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>
        </div>
    </div>

    <div class="sb-editor-shell">
        <div class="sb-editor-column">
            <div class="sb-editor-sticky">
                <div class="sb-panel">
                    <h2 class="sb-panel-title">Страницы</h2>

                    <div class="sb-form-row align-end">
                        <div class="sb-field">
                            <label for="newPageTitle">Новая страница</label>
                            <input class="sb-input" type="text" id="newPageTitle" placeholder="Название страницы">
                        </div>
                        <button class="sb-btn sb-btn-primary" type="button" id="createPageBtn">Создать</button>
                    </div>

                    <div class="sb-divider"></div>

                    <div id="pagesList" class="sb-editor-tree">
                        <div class="sb-editor-empty">Загрузка страниц...</div>
                    </div>
                </div>

                <div class="sb-panel">
                    <h2 class="sb-panel-title">Добавить блок</h2>
                    <div class="sb-toolbar">
                        <button class="sb-btn sb-btn-light" type="button" data-add-block="heading">Заголовок</button>
                        <button class="sb-btn sb-btn-light" type="button" data-add-block="text">Текст</button>
                        <button class="sb-btn sb-btn-light" type="button" data-add-block="button">Кнопка</button>
                        <button class="sb-btn sb-btn-light" type="button" data-add-block="html">HTML</button>
                    </div>
                </div>
            </div>
        </div>

        <div class="sb-editor-column">
            <div class="sb-editor-preview">
                <div class="sb-editor-preview__head">
                    <strong id="previewPageTitle">Страница</strong>
                    <div class="sb-editor-toolbar">
                        <button class="sb-btn sb-btn-light sb-btn-small" type="button" id="movePageUpBtn">Страницу ↑</button>
                        <button class="sb-btn sb-btn-light sb-btn-small" type="button" id="movePageDownBtn">Страницу ↓</button>
                        <button class="sb-btn sb-btn-light sb-btn-small" type="button" id="publishPageBtn">Опубликовать</button>
                    </div>
                </div>
                <div class="sb-editor-preview__body">
                    <div class="sb-editor-page-preview">
                        <h2 class="sb-editor-page-title" id="previewHeading">Выберите страницу</h2>
                        <div id="blocksList">
                            <div class="sb-editor-empty">Список блоков появится здесь</div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="sb-editor-column sb-editor-column--right">
            <div class="sb-editor-sticky">
                <div class="sb-panel">
                    <h2 class="sb-panel-title">Свойства страницы</h2>

                    <div class="sb-editor-form-grid">
                        <div class="sb-field">
                            <label for="pageTitleInput">Название</label>
                            <input class="sb-input" type="text" id="pageTitleInput">
                        </div>

                        <div class="sb-field">
                            <label for="pageSlugInput">Slug</label>
                            <input class="sb-input" type="text" id="pageSlugInput">
                        </div>

                        <div class="sb-field">
                            <label for="pageStatusInput">Статус</label>
                            <select class="sb-select" id="pageStatusInput">
                                <option value="draft">draft</option>
                                <option value="published">published</option>
                            </select>
                        </div>

                        <div class="sb-editor-inspector-actions">
                            <button class="sb-btn sb-btn-primary" type="button" id="savePageBtn">Сохранить страницу</button>
                            <button class="sb-btn sb-btn-danger" type="button" id="deletePageBtn">Удалить страницу</button>
                        </div>
                    </div>
                </div>

                <div class="sb-panel">
                    <h2 class="sb-panel-title">Свойства блока</h2>
                    <div id="blockInspectorEmpty" class="sb-editor-empty">Выберите блок</div>

                    <div id="blockInspector" class="sb-hidden">
                        <div class="sb-editor-form-grid">
                            <div class="sb-field">
                                <label for="blockTypeInput">Тип</label>
                                <input class="sb-input" type="text" id="blockTypeInput" disabled>
                            </div>

                            <div class="sb-field">
                                <label for="blockContentInput">Контент (JSON)</label>
                                <textarea class="sb-textarea" id="blockContentInput"></textarea>
                            </div>

                            <div class="sb-field">
                                <label for="blockPropsInput">Свойства (JSON)</label>
                                <textarea class="sb-textarea" id="blockPropsInput"></textarea>
                            </div>

                            <div class="sb-editor-inspector-actions">
                                <button class="sb-btn sb-btn-primary" type="button" id="saveBlockBtn">Сохранить блок</button>
                                <button class="sb-btn sb-btn-light" type="button" id="duplicateBlockBtn">Дублировать</button>
                                <button class="sb-btn sb-btn-light" type="button" id="moveBlockUpBtn">Блок ↑</button>
                                <button class="sb-btn sb-btn-light" type="button" id="moveBlockDownBtn">Блок ↓</button>
                                <button class="sb-btn sb-btn-danger" type="button" id="deleteBlockBtn">Удалить</button>
                            </div>
                        </div>
                    </div>
                </div>

                <div class="sb-panel">
                    <h2 class="sb-panel-title">Ответ API</h2>
                    <div id="output" class="sb-output">Здесь будут ответы API...</div>
                </div>
            </div>
        </div>
    </div>
</div>

<script src="/bitrix/js/main/core/core.js"></script>
<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var siteId = <?= (int)$siteId ?>;

    var state = {
        site: null,
        pages: [],
        currentPageId: 0,
        blocks: [],
        currentBlockId: 0
    };

    var output = document.getElementById('output');
    var pagesList = document.getElementById('pagesList');
    var blocksList = document.getElementById('blocksList');

    function print(data) {
        try {
            output.textContent = typeof data === 'string' ? data : JSON.stringify(data, null, 2);
        } catch (e) {
            output.textContent = String(data);
        }
    }

    function escapeHtml(value) {
        return String(value == null ? '' : value)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#039;');
    }

    function getSessid() {
        if (window.BX && typeof BX.bitrix_sessid === 'function') {
            return BX.bitrix_sessid();
        }
        return '';
    }

    function api(action, data) {
        return new Promise(function (resolve, reject) {
            BX.ajax({
                url: API_URL,
                method: 'POST',
                dataType: 'json',
                timeout: 60,
                data: Object.assign({
                    action: action,
                    sessid: getSessid()
                }, data || {}),
                onsuccess: function (res) {
                    print(res);
                    if (res && res.ok) {
                        resolve(res);
                    } else {
                        reject(res || {error: 'UNKNOWN'});
                    }
                },
                onfailure: function (err) {
                    reject(err);
                }
            });
        });
    }

    async function loadSite() {
        var res = await api('site.get', {siteId: siteId});
        state.site = res.site || null;

        document.getElementById('siteTitle').textContent = state.site ? state.site.name : 'Редактор сайта';
        document.getElementById('siteSubtitle').textContent = state.site
            ? ('ID: ' + state.site.id + ' · slug: ' + (state.site.slug || '') + ' · homePageId: ' + (state.site.homePageId || 0))
            : 'Сайт не найден';
    }

    async function loadPages() {
        var res = await api('page.list', {siteId: siteId});
        state.pages = Array.isArray(res.pages) ? res.pages : [];

        if (!state.currentPageId && state.pages.length) {
            state.currentPageId = Number(state.pages[0].id || 0);
        }

        renderPages();
        fillPageForm();
    }

    async function loadBlocks() {
        if (!state.currentPageId) {
            state.blocks = [];
            state.currentBlockId = 0;
            renderBlocks();
            fillBlockForm();
            return;
        }

        var res = await api('block.list', {pageId: state.currentPageId});
        state.blocks = Array.isArray(res.blocks) ? res.blocks : [];

        if (state.currentBlockId) {
            var exists = state.blocks.some(function (b) {
                return Number(b.id || 0) === state.currentBlockId;
            });
            if (!exists) {
                state.currentBlockId = 0;
            }
        }

        renderBlocks();
        fillBlockForm();
    }

    function renderPages() {
        if (!state.pages.length) {
            pagesList.innerHTML = '<div class="sb-editor-empty">Страниц пока нет</div>';
            return;
        }

        pagesList.innerHTML = state.pages.map(function (page) {
            var active = Number(page.id || 0) === state.currentPageId ? ' is-active' : '';
            return ''
                + '<div class="sb-editor-page-item' + active + '" data-page-id="' + Number(page.id || 0) + '">'
                + '  <div class="sb-editor-page-item__title">' + escapeHtml(page.title || '') + '</div>'
                + '  <div class="sb-editor-page-item__meta">'
                +       'slug: ' + escapeHtml(page.slug || '') + ' · status: ' + escapeHtml(page.status || 'draft')
                + '  </div>'
                + '</div>';
        }).join('');
    }

    function renderBlocks() {
        var page = getCurrentPage();
        document.getElementById('previewHeading').textContent = page ? (page.title || 'Страница') : 'Выберите страницу';
        document.getElementById('previewPageTitle').textContent = page ? (page.title || 'Страница') : 'Страница';

        if (!state.currentPageId) {
            blocksList.innerHTML = '<div class="sb-editor-empty">Сначала выберите страницу</div>';
            return;
        }

        if (!state.blocks.length) {
            blocksList.innerHTML = '<div class="sb-editor-empty">На странице пока нет блоков</div>';
            return;
        }

        blocksList.innerHTML = state.blocks.map(function (block) {
            var active = Number(block.id || 0) === state.currentBlockId ? ' is-active' : '';
            return ''
                + '<div class="sb-editor-block' + active + '" data-block-id="' + Number(block.id || 0) + '">'
                + '  <div class="sb-editor-block__meta">'
                + '      <span class="sb-editor-block__type">' + escapeHtml(block.type || 'block') + '</span>'
                + '      <div class="sb-editor-block__actions">'
                + '          <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-select-block="' + Number(block.id || 0) + '">Выбрать</button>'
                + '      </div>'
                + '  </div>'
                + '  <div class="sb-editor-block__content">' + escapeHtml(previewBlockText(block)) + '</div>'
                + '</div>';
        }).join('');
    }

    function previewBlockText(block) {
        var type = String(block.type || '');
        var content = block.content || {};

        if (type === 'heading') {
            return content.text || '[пустой заголовок]';
        }
        if (type === 'text') {
            return content.text || '[пустой текст]';
        }
        if (type === 'button') {
            return (content.label || 'Кнопка') + (content.href ? ' → ' + content.href : '');
        }
        if (type === 'html') {
            return (content.html || '').slice(0, 180) || '[пустой HTML]';
        }

        try {
            return JSON.stringify(content);
        } catch (e) {
            return '[контент блока]';
        }
    }

    function getCurrentPage() {
        return state.pages.find(function (page) {
            return Number(page.id || 0) === state.currentPageId;
        }) || null;
    }

    function getCurrentBlock() {
        return state.blocks.find(function (block) {
            return Number(block.id || 0) === state.currentBlockId;
        }) || null;
    }

    function fillPageForm() {
        var page = getCurrentPage();

        document.getElementById('pageTitleInput').value = page ? (page.title || '') : '';
        document.getElementById('pageSlugInput').value = page ? (page.slug || '') : '';
        document.getElementById('pageStatusInput').value = page ? (page.status || 'draft') : 'draft';
    }

    function fillBlockForm() {
        var block = getCurrentBlock();
        var emptyNode = document.getElementById('blockInspectorEmpty');
        var formNode = document.getElementById('blockInspector');

        if (!block) {
            emptyNode.classList.remove('sb-hidden');
            formNode.classList.add('sb-hidden');
            document.getElementById('blockTypeInput').value = '';
            document.getElementById('blockContentInput').value = '';
            document.getElementById('blockPropsInput').value = '';
            return;
        }

        emptyNode.classList.add('sb-hidden');
        formNode.classList.remove('sb-hidden');

        document.getElementById('blockTypeInput').value = block.type || '';
        document.getElementById('blockContentInput').value = JSON.stringify(block.content || {}, null, 2);
        document.getElementById('blockPropsInput').value = JSON.stringify(block.props || {}, null, 2);
    }

    async function createPage() {
        var title = (document.getElementById('newPageTitle').value || '').trim();
        if (!title) {
            alert('Введите название страницы');
            return;
        }

        await api('page.create', {
            siteId: siteId,
            title: title
        });

        document.getElementById('newPageTitle').value = '';
        await loadPages();
        await loadBlocks();
    }

    async function savePage() {
        if (!state.currentPageId) return;

        await api('page.updateMeta', {
            id: state.currentPageId,
            title: document.getElementById('pageTitleInput').value.trim(),
            slug: document.getElementById('pageSlugInput').value.trim()
        });

        await api('page.setStatus', {
            id: state.currentPageId,
            status: document.getElementById('pageStatusInput').value
        });

        await loadPages();
        await loadBlocks();
    }

    async function deletePage() {
        if (!state.currentPageId) return;
        if (!confirm('Удалить страницу?')) return;

        var idToDelete = state.currentPageId;
        await api('page.delete', {id: idToDelete});

        if (state.currentPageId === idToDelete) {
            state.currentPageId = 0;
        }

        await loadPages();
        await loadBlocks();
    }

    async function movePage(dir) {
        if (!state.currentPageId) return;
        await api('page.move', {id: state.currentPageId, dir: dir});
        await loadPages();
    }

    async function createBlock(type) {
        if (!state.currentPageId) {
            alert('Сначала выберите страницу');
            return;
        }

        await api('block.create', {
            pageId: state.currentPageId,
            type: type
        });

        await loadBlocks();
    }

    async function saveBlock() {
        var block = getCurrentBlock();
        if (!block) return;

        var content;
        var props;

        try {
            content = JSON.parse(document.getElementById('blockContentInput').value || '{}');
        } catch (e) {
            alert('Контент блока должен быть валидным JSON');
            return;
        }

        try {
            props = JSON.parse(document.getElementById('blockPropsInput').value || '{}');
        } catch (e) {
            alert('Свойства блока должны быть валидным JSON');
            return;
        }

        await api('block.update', {
            id: block.id,
            content: JSON.stringify(content),
            props: JSON.stringify(props)
        });

        await loadBlocks();
    }

    async function duplicateBlock() {
        var block = getCurrentBlock();
        if (!block) return;

        await api('block.duplicate', {id: block.id});
        await loadBlocks();
    }

    async function deleteBlock() {
        var block = getCurrentBlock();
        if (!block) return;
        if (!confirm('Удалить блок?')) return;

        await api('block.delete', {id: block.id});
        state.currentBlockId = 0;
        await loadBlocks();
    }

    async function moveBlock(dir) {
        var block = getCurrentBlock();
        if (!block) return;

        await api('block.move', {
            id: block.id,
            dir: dir
        });

        await loadBlocks();
    }

    pagesList.addEventListener('click', async function (e) {
        var item = e.target.closest('[data-page-id]');
        if (!item) return;

        state.currentPageId = Number(item.getAttribute('data-page-id') || 0);
        state.currentBlockId = 0;
        renderPages();
        fillPageForm();
        await loadBlocks();
    });

    blocksList.addEventListener('click', function (e) {
        var item = e.target.closest('[data-block-id]');
        if (!item) return;

        state.currentBlockId = Number(item.getAttribute('data-block-id') || 0);
        renderBlocks();
        fillBlockForm();
    });

    document.getElementById('createPageBtn').addEventListener('click', createPage);
    document.getElementById('savePageBtn').addEventListener('click', savePage);
    document.getElementById('deletePageBtn').addEventListener('click', deletePage);
    document.getElementById('movePageUpBtn').addEventListener('click', function () { movePage('up'); });
    document.getElementById('movePageDownBtn').addEventListener('click', function () { movePage('down'); });
    document.getElementById('publishPageBtn').addEventListener('click', async function () {
        if (!state.currentPageId) return;
        await api('page.setStatus', {id: state.currentPageId, status: 'published'});
        await loadPages();
    });

    document.querySelectorAll('[data-add-block]').forEach(function (btn) {
        btn.addEventListener('click', function () {
            createBlock(btn.getAttribute('data-add-block'));
        });
    });

    document.getElementById('saveBlockBtn').addEventListener('click', saveBlock);
    document.getElementById('duplicateBlockBtn').addEventListener('click', duplicateBlock);
    document.getElementById('deleteBlockBtn').addEventListener('click', deleteBlock);
    document.getElementById('moveBlockUpBtn').addEventListener('click', function () { moveBlock('up'); });
    document.getElementById('moveBlockDownBtn').addEventListener('click', function () { moveBlock('down'); });

    window.onerror = function (message, source, lineno, colno, error) {
        print({
            jsError: true,
            message: message,
            source: source,
            line: lineno,
            column: colno,
            stack: error && error.stack ? error.stack : null
        });
    };

    (async function init() {
        try {
            await loadSite();
            await loadPages();
            await loadBlocks();
        } catch (e) {
            print(e);
            alert('Не удалось загрузить редактор');
        }
    })();
})();
</script>
</body>
</html>
