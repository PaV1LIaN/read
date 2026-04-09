Отлично. Делаем первый рабочий editor.php.

Что он будет уметь уже сейчас:

загружать сайт

показывать список страниц сайта

создавать страницу

удалять страницу

дублировать страницу

менять статус страницы

выбирать текущую страницу

загружать блоки текущей страницы

добавлять блоки

удалять блоки

дублировать блоки

двигать блоки вверх/вниз

редактировать базовые типы:

text

heading

button

html



Без сложных модалок и без layout-части. Но это уже будет живой редактор на новом API.


---

/local/sitebuilder/editor.php

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

global $APPLICATION, $USER;

if (!$USER->IsAuthorized()) {
    require $_SERVER['DOCUMENT_ROOT'] . '/auth.php';
    exit;
}

CJSCore::Init(['ajax']);

header('Content-Type: text/html; charset=UTF-8');

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$siteId = (int)($_GET['siteId'] ?? 0);

if ($siteId <= 0) {
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>SiteBuilder / Editor</title>
        <?php $APPLICATION->ShowHead(); ?>
    </head>
    <body style="font-family:Arial,sans-serif;padding:20px;">
        <h1>Не передан siteId</h1>
        <p><a href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">Вернуться к списку сайтов</a></p>
    </body>
    </html>
    <?php
    exit;
}
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder / Editor</title>
    <?php $APPLICATION->ShowHead(); ?>
    <style>
        * { box-sizing: border-box; }
        body {
            margin: 0;
            font-family: Arial, sans-serif;
            background: #f6f8fb;
            color: #1f2937;
        }
        .page {
            max-width: 1440px;
            margin: 0 auto;
            padding: 20px;
        }
        .topbar {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 16px;
            margin-bottom: 20px;
        }
        .topbar-left h1 {
            margin: 0;
            font-size: 28px;
            font-weight: 700;
        }
        .subtitle {
            margin: 6px 0 0;
            color: #6b7280;
            font-size: 14px;
        }
        .back-link {
            display: inline-block;
            margin-bottom: 8px;
            text-decoration: none;
            color: #1d4ed8;
            font-size: 14px;
        }
        .layout {
            display: grid;
            grid-template-columns: 360px 1fr;
            gap: 20px;
        }
        .panel {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 16px;
            box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
        }
        .panel-title {
            margin: 0 0 14px;
            font-size: 18px;
            font-weight: 700;
        }
        .form-row {
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
            align-items: end;
        }
        .field {
            display: flex;
            flex-direction: column;
            gap: 6px;
            min-width: 180px;
            flex: 1 1 180px;
        }
        .field label {
            font-size: 13px;
            color: #4b5563;
        }
        .field input,
        .field select,
        .field textarea {
            width: 100%;
            border: 1px solid #d1d5db;
            border-radius: 10px;
            outline: none;
            background: #fff;
            padding: 10px 12px;
            font: inherit;
        }
        .field input,
        .field select {
            height: 40px;
        }
        .field textarea {
            min-height: 120px;
            resize: vertical;
        }
        .field input:focus,
        .field select:focus,
        .field textarea:focus {
            border-color: #2563eb;
        }
        .btn {
            height: 40px;
            border: 0;
            border-radius: 10px;
            padding: 0 16px;
            cursor: pointer;
            font-weight: 600;
        }
        .btn-small {
            height: 34px;
            padding: 0 12px;
            font-size: 13px;
        }
        .btn-primary {
            background: #2563eb;
            color: #fff;
        }
        .btn-primary:hover {
            background: #1d4ed8;
        }
        .btn-light {
            background: #eef2ff;
            color: #1e3a8a;
        }
        .btn-light:hover {
            background: #e0e7ff;
        }
        .btn-danger {
            background: #dc2626;
            color: #fff;
        }
        .btn-danger:hover {
            background: #b91c1c;
        }
        .btn-gray {
            background: #f3f4f6;
            color: #374151;
        }
        .btn-gray:hover {
            background: #e5e7eb;
        }
        .pages-list,
        .blocks-list {
            display: flex;
            flex-direction: column;
            gap: 12px;
        }
        .page-card,
        .block-card {
            border: 1px solid #e5e7eb;
            border-radius: 12px;
            padding: 12px;
            background: #fafafa;
        }
        .page-card.active {
            border-color: #2563eb;
            background: #eff6ff;
        }
        .page-head,
        .block-head {
            display: flex;
            justify-content: space-between;
            gap: 12px;
            align-items: flex-start;
        }
        .page-title,
        .block-title {
            margin: 0 0 6px;
            font-size: 15px;
            font-weight: 700;
        }
        .meta {
            font-size: 13px;
            color: #6b7280;
            line-height: 1.5;
        }
        .actions {
            margin-top: 10px;
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
        }
        .badge {
            display: inline-flex;
            align-items: center;
            background: #f3f4f6;
            color: #374151;
            border-radius: 999px;
            padding: 4px 10px;
            font-size: 12px;
            white-space: nowrap;
        }
        .badge-green {
            background: #dcfce7;
            color: #166534;
        }
        .badge-yellow {
            background: #fef3c7;
            color: #92400e;
        }
        .block-preview {
            margin-top: 12px;
            padding: 12px;
            border-radius: 10px;
            background: #fff;
            border: 1px solid #e5e7eb;
        }
        .toolbar {
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
            margin-bottom: 14px;
        }
        .empty {
            padding: 20px;
            text-align: center;
            color: #6b7280;
            border: 1px dashed #d1d5db;
            border-radius: 12px;
            background: #fff;
        }
        .editor-grid {
            display: grid;
            grid-template-columns: 1fr 420px;
            gap: 20px;
        }
        .output {
            white-space: pre-wrap;
            background: #0f172a;
            color: #e5e7eb;
            border-radius: 12px;
            padding: 14px;
            min-height: 120px;
            font-family: Consolas, Monaco, monospace;
            font-size: 13px;
            overflow: auto;
        }
        .hidden {
            display: none !important;
        }
        @media (max-width: 1200px) {
            .layout,
            .editor-grid {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
<div class="page">
    <div class="topbar">
        <div class="topbar-left">
            <a class="back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← К списку сайтов</a>
            <h1>Редактор сайта</h1>
            <p class="subtitle">siteId = <?= (int)$siteId ?></p>
        </div>
    </div>

    <div class="layout">
        <div class="panel">
            <h2 class="panel-title">Страницы</h2>

            <div class="form-row" style="margin-bottom:14px;">
                <div class="field">
                    <label for="newPageTitle">Название страницы</label>
                    <input type="text" id="newPageTitle" placeholder="Например: Главная">
                </div>
                <div class="field">
                    <label for="newPageSlug">Slug</label>
                    <input type="text" id="newPageSlug" placeholder="Например: home">
                </div>
                <button type="button" class="btn btn-primary" id="createPageBtn">Создать</button>
            </div>

            <div id="pagesContainer" class="pages-list">
                <div class="empty">Загрузка страниц...</div>
            </div>
        </div>

        <div class="editor-grid">
            <div class="panel">
                <div style="display:flex; justify-content:space-between; align-items:center; gap:12px; margin-bottom:14px;">
                    <h2 class="panel-title" style="margin:0;">Блоки страницы</h2>
                    <button type="button" class="btn btn-light btn-small" id="reloadBlocksBtn">Обновить</button>
                </div>

                <div id="currentPageInfo" class="meta" style="margin-bottom:14px;">
                    Страница не выбрана
                </div>

                <div class="toolbar">
                    <button type="button" class="btn btn-light btn-small js-add-block" data-type="text">+ Text</button>
                    <button type="button" class="btn btn-light btn-small js-add-block" data-type="heading">+ Heading</button>
                    <button type="button" class="btn btn-light btn-small js-add-block" data-type="button">+ Button</button>
                    <button type="button" class="btn btn-light btn-small js-add-block" data-type="html">+ HTML</button>
                    <button type="button" class="btn btn-light btn-small js-add-block" data-type="spacer">+ Spacer</button>
                </div>

                <div id="blocksContainer" class="blocks-list">
                    <div class="empty">Выберите страницу</div>
                </div>
            </div>

            <div class="panel">
                <h2 class="panel-title">Редактор блока</h2>

                <div id="blockEditorEmpty" class="empty">Выберите блок для редактирования</div>

                <div id="blockEditorForm" class="hidden">
                    <div class="field" style="margin-bottom:12px;">
                        <label for="editBlockType">Тип</label>
                        <input type="text" id="editBlockType" readonly>
                    </div>

                    <div class="field" style="margin-bottom:12px;">
                        <label for="editBlockContentText">Content / JSON</label>
                        <textarea id="editBlockContentText"></textarea>
                    </div>

                    <div class="field" style="margin-bottom:12px;">
                        <label for="editBlockPropsText">Props / JSON</label>
                        <textarea id="editBlockPropsText">{}</textarea>
                    </div>

                    <div class="form-row">
                        <button type="button" class="btn btn-primary" id="saveBlockBtn">Сохранить блок</button>
                        <button type="button" class="btn btn-danger" id="deleteBlockBtn">Удалить блок</button>
                    </div>
                </div>
            </div>
        </div>
    </div>

    <div class="panel" style="margin-top:20px;">
        <h2 class="panel-title">Отладка</h2>
        <div id="output" class="output">Здесь будут ответы API...</div>
    </div>
</div>

<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var SITE_ID = <?= (int)$siteId ?>;

    var output = document.getElementById('output');
    var pagesContainer = document.getElementById('pagesContainer');
    var blocksContainer = document.getElementById('blocksContainer');
    var currentPageInfo = document.getElementById('currentPageInfo');

    var state = {
        site: null,
        pages: [],
        blocks: [],
        currentPageId: 0,
        currentBlockId: 0
    };

    function print(data) {
        if (typeof data === 'string') {
            output.textContent = data;
            return;
        }
        try {
            output.textContent = JSON.stringify(data, null, 2);
        } catch (e) {
            output.textContent = String(data);
        }
    }

    function escapeHtml(value) {
        return String(value)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#039;');
    }

    function getSessid() {
        if (typeof window.BX !== 'undefined' && typeof BX.bitrix_sessid === 'function') {
            return BX.bitrix_sessid();
        }
        return '<?= CUtil::JSEscape(bitrix_sessid()) ?>';
    }

    function api(action, data, onSuccess, onFailure) {
        if (typeof window.BX === 'undefined' || typeof BX.ajax !== 'function') {
            print('BX.ajax не загружен');
            return;
        }

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
                if (typeof onSuccess === 'function') {
                    onSuccess(res);
                }
            },
            onfailure: function (err) {
                print({
                    ok: false,
                    error: 'AJAX_ERROR',
                    detail: err
                });
                if (typeof onFailure === 'function') {
                    onFailure(err);
                }
            }
        });
    }

    function loadSite(next) {
        api('site.get', { siteId: SITE_ID }, function (res) {
            if (res && res.ok === true) {
                state.site = res.site || null;
            }
            if (typeof next === 'function') {
                next();
            }
        });
    }

    function loadPages(next) {
        pagesContainer.innerHTML = '<div class="empty">Загрузка страниц...</div>';

        api('page.list', { siteId: SITE_ID }, function (res) {
            if (!res || res.ok !== true) {
                pagesContainer.innerHTML = '<div class="empty">Не удалось загрузить страницы</div>';
                return;
            }

            state.pages = Array.isArray(res.pages) ? res.pages : [];

            if (!state.currentPageId && state.pages.length) {
                state.currentPageId = Number(state.pages[0].id || 0);
            }

            var hasCurrent = false;
            for (var i = 0; i < state.pages.length; i++) {
                if (Number(state.pages[i].id || 0) === Number(state.currentPageId || 0)) {
                    hasCurrent = true;
                    break;
                }
            }

            if (!hasCurrent) {
                state.currentPageId = state.pages.length ? Number(state.pages[0].id || 0) : 0;
            }

            renderPages();

            if (typeof next === 'function') {
                next();
            }
        });
    }

    function renderPages() {
        if (!state.pages.length) {
            pagesContainer.innerHTML = '<div class="empty">Страниц пока нет</div>';
            currentPageInfo.textContent = 'Страница не выбрана';
            blocksContainer.innerHTML = '<div class="empty">Выберите страницу</div>';
            return;
        }

        var html = '';
        for (var i = 0; i < state.pages.length; i++) {
            html += renderPageCard(state.pages[i]);
        }
        pagesContainer.innerHTML = html;
    }

    function renderPageCard(page) {
        var id = Number(page.id || 0);
        var active = id === Number(state.currentPageId || 0) ? ' active' : '';
        var isPublished = String(page.status || '') === 'published';
        var badge = isPublished
            ? '<span class="badge badge-green">published</span>'
            : '<span class="badge badge-yellow">draft</span>';

        return ''
            + '<div class="page-card' + active + '">'
            + '  <div class="page-head">'
            + '    <div>'
            + '      <div class="page-title">' + escapeHtml(page.title || '') + '</div>'
            + '      <div class="meta">'
            + '        <div><strong>ID:</strong> ' + id + '</div>'
            + '        <div><strong>Slug:</strong> ' + escapeHtml(page.slug || '') + '</div>'
            + '        <div><strong>Parent ID:</strong> ' + Number(page.parentId || 0) + '</div>'
            + '        <div><strong>Sort:</strong> ' + Number(page.sort || 0) + '</div>'
            + '      </div>'
            + '    </div>'
            + '    ' + badge
            + '  </div>'
            + '  <div class="actions">'
            + '    <button type="button" class="btn btn-light btn-small js-open-page" data-id="' + id + '">Открыть</button>'
            + '    <button type="button" class="btn btn-light btn-small js-rename-page" data-id="' + id + '">Meta</button>'
            + '    <button type="button" class="btn btn-light btn-small js-duplicate-page" data-id="' + id + '">Дублировать</button>'
            + '    <button type="button" class="btn btn-light btn-small js-toggle-status" data-id="' + id + '" data-status="' + escapeHtml(page.status || 'draft') + '">'
            + (isPublished ? 'В draft' : 'Опубликовать')
            + '    </button>'
            + '    <button type="button" class="btn btn-gray btn-small js-move-page-up" data-id="' + id + '">↑</button>'
            + '    <button type="button" class="btn btn-gray btn-small js-move-page-down" data-id="' + id + '">↓</button>'
            + '    <button type="button" class="btn btn-danger btn-small js-delete-page" data-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }

    function createPage() {
        var titleInput = document.getElementById('newPageTitle');
        var slugInput = document.getElementById('newPageSlug');

        var title = (titleInput.value || '').trim();
        var slug = (slugInput.value || '').trim();

        if (!title) {
            alert('Введите название страницы');
            titleInput.focus();
            return;
        }

        api('page.create', {
            siteId: SITE_ID,
            title: title,
            slug: slug
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось создать страницу');
                return;
            }

            titleInput.value = '';
            slugInput.value = '';

            state.currentPageId = Number((res.page && res.page.id) || 0);
            loadPages(loadBlocks);
        });
    }

    function openPage(pageId) {
        state.currentPageId = Number(pageId || 0);
        state.currentBlockId = 0;
        renderPages();
        loadBlocks();
        clearBlockEditor();
    }

    function renamePage(pageId) {
        var page = findPage(pageId);
        if (!page) {
            alert('Страница не найдена');
            return;
        }

        var title = prompt('Название страницы:', page.title || '');
        if (title === null) {
            return;
        }
        title = title.trim();
        if (!title) {
            alert('Название не может быть пустым');
            return;
        }

        var slug = prompt('Slug страницы:', page.slug || '');
        if (slug === null) {
            return;
        }
        slug = slug.trim();

        api('page.updateMeta', {
            id: pageId,
            title: title,
            slug: slug
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось обновить страницу');
                return;
            }

            loadPages();
        });
    }

    function duplicatePage(pageId) {
        api('page.duplicate', { id: pageId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось дублировать страницу');
                return;
            }

            state.currentPageId = Number((res.page && res.page.id) || 0);
            loadPages(loadBlocks);
        });
    }

    function togglePageStatus(pageId, currentStatus) {
        var nextStatus = currentStatus === 'published' ? 'draft' : 'published';

        api('page.setStatus', {
            id: pageId,
            status: nextStatus
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось изменить статус');
                return;
            }

            loadPages();
        });
    }

    function movePage(pageId, dir) {
        api('page.move', {
            id: pageId,
            dir: dir
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось переместить страницу');
                return;
            }

            loadPages();
        });
    }

    function deletePage(pageId) {
        if (!confirm('Удалить страницу #' + pageId + '?')) {
            return;
        }

        api('page.delete', { id: pageId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось удалить страницу');
                return;
            }

            if (Number(state.currentPageId || 0) === Number(pageId || 0)) {
                state.currentPageId = 0;
                state.currentBlockId = 0;
                clearBlockEditor();
            }

            loadPages(loadBlocks);
        });
    }

    function loadBlocks() {
        if (!state.currentPageId) {
            currentPageInfo.textContent = 'Страница не выбрана';
            blocksContainer.innerHTML = '<div class="empty">Выберите страницу</div>';
            return;
        }

        var page = findPage(state.currentPageId);
        if (page) {
            currentPageInfo.innerHTML =
                '<strong>Текущая страница:</strong> '
                + escapeHtml(page.title || '')
                + ' (#' + Number(page.id || 0) + ', slug: ' + escapeHtml(page.slug || '') + ')';
        }

        blocksContainer.innerHTML = '<div class="empty">Загрузка блоков...</div>';

        api('block.list', { pageId: state.currentPageId }, function (res) {
            if (!res || res.ok !== true) {
                blocksContainer.innerHTML = '<div class="empty">Не удалось загрузить блоки</div>';
                return;
            }

            state.blocks = Array.isArray(res.blocks) ? res.blocks : [];
            renderBlocks();
        });
    }

    function renderBlocks() {
        if (!state.blocks.length) {
            blocksContainer.innerHTML = '<div class="empty">Блоков пока нет</div>';
            return;
        }

        var html = '';
        for (var i = 0; i < state.blocks.length; i++) {
            html += renderBlockCard(state.blocks[i], i);
        }
        blocksContainer.innerHTML = html;
    }

    function renderBlockCard(block, index) {
        var id = Number(block.id || 0);
        var type = escapeHtml(block.type || '');
        var preview = escapeHtml(buildPreviewText(block));

        return ''
            + '<div class="block-card">'
            + '  <div class="block-head">'
            + '    <div>'
            + '      <div class="block-title">' + type + ' #' + id + '</div>'
            + '      <div class="meta">'
            + '        <div><strong>Sort:</strong> ' + Number(block.sort || 0) + '</div>'
            + '      </div>'
            + '    </div>'
            + '    <span class="badge">block</span>'
            + '  </div>'
            + '  <div class="block-preview">' + preview + '</div>'
            + '  <div class="actions">'
            + '    <button type="button" class="btn btn-light btn-small js-edit-block" data-id="' + id + '">Редактировать</button>'
            + '    <button type="button" class="btn btn-light btn-small js-duplicate-block" data-id="' + id + '">Дублировать</button>'
            + '    <button type="button" class="btn btn-gray btn-small js-move-block-up" data-id="' + id + '"' + (index === 0 ? ' disabled' : '') + '>↑</button>'
            + '    <button type="button" class="btn btn-gray btn-small js-move-block-down" data-id="' + id + '"' + (index === state.blocks.length - 1 ? ' disabled' : '') + '>↓</button>'
            + '    <button type="button" class="btn btn-danger btn-small js-delete-block" data-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }

    function buildPreviewText(block) {
        var type = String(block.type || '');
        var content = block.content || {};

        if (type === 'text') {
            return String(content.html || '').replace(/<[^>]*>/g, ' ').trim() || '[text]';
        }
        if (type === 'heading') {
            return String(content.text || '') || '[heading]';
        }
        if (type === 'button') {
            return (content.text || '[button]') + ' → ' + (content.href || '');
        }
        if (type === 'html') {
            return String(content.html || '').replace(/<[^>]*>/g, ' ').trim() || '[html]';
        }
        if (type === 'spacer') {
            return 'height: ' + String(content.height || 0);
        }

        return JSON.stringify(content);
    }

    function addBlock(type) {
        if (!state.currentPageId) {
            alert('Сначала выберите страницу');
            return;
        }

        api('block.create', {
            pageId: state.currentPageId,
            type: type
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось создать блок');
                return;
            }

            loadBlocks();
        });
    }

    function editBlock(blockId) {
        var block = findBlock(blockId);
        if (!block) {
            alert('Блок не найден');
            return;
        }

        state.currentBlockId = Number(blockId || 0);

        document.getElementById('blockEditorEmpty').classList.add('hidden');
        document.getElementById('blockEditorForm').classList.remove('hidden');
        document.getElementById('editBlockType').value = block.type || '';
        document.getElementById('editBlockContentText').value = JSON.stringify(block.content || {}, null, 2);
        document.getElementById('editBlockPropsText').value = JSON.stringify(block.props || {}, null, 2);
    }

    function clearBlockEditor() {
        state.currentBlockId = 0;
        document.getElementById('blockEditorEmpty').classList.remove('hidden');
        document.getElementById('blockEditorForm').classList.add('hidden');
        document.getElementById('editBlockType').value = '';
        document.getElementById('editBlockContentText').value = '';
        document.getElementById('editBlockPropsText').value = '{}';
    }

    function saveBlock() {
        if (!state.currentBlockId) {
            return;
        }

        var contentText = document.getElementById('editBlockContentText').value || '{}';
        var propsText = document.getElementById('editBlockPropsText').value || '{}';

        try {
            JSON.parse(contentText);
        } catch (e) {
            alert('Content должен быть валидным JSON');
            return;
        }

        try {
            JSON.parse(propsText);
        } catch (e) {
            alert('Props должен быть валидным JSON');
            return;
        }

        api('block.update', {
            id: state.currentBlockId,
            content: contentText,
            props: propsText
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось сохранить блок');
                return;
            }

            loadBlocks();
        });
    }

    function deleteBlock(blockId) {
        if (!confirm('Удалить блок #' + blockId + '?')) {
            return;
        }

        api('block.delete', { id: blockId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось удалить блок');
                return;
            }

            if (Number(state.currentBlockId || 0) === Number(blockId || 0)) {
                clearBlockEditor();
            }

            loadBlocks();
        });
    }

    function duplicateBlock(blockId) {
        api('block.duplicate', { id: blockId }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось дублировать блок');
                return;
            }

            loadBlocks();
        });
    }

    function moveBlock(blockId, dir) {
        api('block.move', {
            id: blockId,
            dir: dir
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось переместить блок');
                return;
            }

            loadBlocks();
        });
    }

    function findPage(pageId) {
        for (var i = 0; i < state.pages.length; i++) {
            if (Number(state.pages[i].id || 0) === Number(pageId || 0)) {
                return state.pages[i];
            }
        }
        return null;
    }

    function findBlock(blockId) {
        for (var i = 0; i < state.blocks.length; i++) {
            if (Number(state.blocks[i].id || 0) === Number(blockId || 0)) {
                return state.blocks[i];
            }
        }
        return null;
    }

    document.getElementById('createPageBtn').addEventListener('click', createPage);
    document.getElementById('reloadBlocksBtn').addEventListener('click', loadBlocks);
    document.getElementById('saveBlockBtn').addEventListener('click', saveBlock);
    document.getElementById('deleteBlockBtn').addEventListener('click', function () {
        if (state.currentBlockId) {
            deleteBlock(state.currentBlockId);
        }
    });

    document.addEventListener('click', function (e) {
        var addBlockBtn = e.target.closest('.js-add-block');
        if (addBlockBtn) {
            addBlock(addBlockBtn.getAttribute('data-type') || 'text');
            return;
        }

        var openPageBtn = e.target.closest('.js-open-page');
        if (openPageBtn) {
            openPage(parseInt(openPageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var renamePageBtn = e.target.closest('.js-rename-page');
        if (renamePageBtn) {
            renamePage(parseInt(renamePageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var duplicatePageBtn = e.target.closest('.js-duplicate-page');
        if (duplicatePageBtn) {
            duplicatePage(parseInt(duplicatePageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var toggleStatusBtn = e.target.closest('.js-toggle-status');
        if (toggleStatusBtn) {
            togglePageStatus(
                parseInt(toggleStatusBtn.getAttribute('data-id'), 10) || 0,
                toggleStatusBtn.getAttribute('data-status') || 'draft'
            );
            return;
        }

        var movePageUpBtn = e.target.closest('.js-move-page-up');
        if (movePageUpBtn) {
            movePage(parseInt(movePageUpBtn.getAttribute('data-id'), 10) || 0, 'up');
            return;
        }

        var movePageDownBtn = e.target.closest('.js-move-page-down');
        if (movePageDownBtn) {
            movePage(parseInt(movePageDownBtn.getAttribute('data-id'), 10) || 0, 'down');
            return;
        }

        var deletePageBtn = e.target.closest('.js-delete-page');
        if (deletePageBtn) {
            deletePage(parseInt(deletePageBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var editBlockBtn = e.target.closest('.js-edit-block');
        if (editBlockBtn) {
            editBlock(parseInt(editBlockBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var deleteBlockBtn = e.target.closest('.js-delete-block');
        if (deleteBlockBtn) {
            deleteBlock(parseInt(deleteBlockBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var duplicateBlockBtn = e.target.closest('.js-duplicate-block');
        if (duplicateBlockBtn) {
            duplicateBlock(parseInt(duplicateBlockBtn.getAttribute('data-id'), 10) || 0);
            return;
        }

        var moveBlockUpBtn = e.target.closest('.js-move-block-up');
        if (moveBlockUpBtn) {
            moveBlock(parseInt(moveBlockUpBtn.getAttribute('data-id'), 10) || 0, 'up');
            return;
        }

        var moveBlockDownBtn = e.target.closest('.js-move-block-down');
        if (moveBlockDownBtn) {
            moveBlock(parseInt(moveBlockDownBtn.getAttribute('data-id'), 10) || 0, 'down');
            return;
        }
    });

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

    loadSite(function () {
        loadPages(function () {
            if (state.currentPageId) {
                loadBlocks();
            }
        });
    });
})();
</script>
</body>
</html>

Что проверить

Открывай:

/local/sitebuilder/editor.php?siteId=1

И проверь по порядку:

1. грузится список страниц


2. создаётся страница


3. выбирается страница


4. добавляется блок text


5. добавляется блок heading


6. редактируется JSON content/props


7. дублируется блок


8. двигается вверх/вниз


9. удаляется блок


10. удаляется/дублируется страница



Что это уже даёт

Это уже не заглушка, а рабочий черновой редактор на новом API.
Да, пока без красивых диалогов, но backend и основной поток редактирования уже начинают жить.

Что логично делать следующим

Теперь после проверки editor.php есть два хороших пути:

сделать files.php

сделать settings.php


Я бы следующим сделал files.php, чтобы сразу добить проверку file.list / file.upload / file.delete и понять, всё ли хорошо с Disk.

Напиши: давай files.php