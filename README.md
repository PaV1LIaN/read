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
        <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    </head>
    <body class="sb-admin-body">
    <div class="sb-page">
        <h1 class="sb-title">Не передан siteId</h1>
        <p><a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">Вернуться к списку сайтов</a></p>
    </div>
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
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    <style>
        .sb-editor-shell {
            display: grid;
            grid-template-columns: 320px minmax(0, 1fr) 360px;
            gap: 20px;
            align-items: start;
        }

        .sb-editor-col {
            min-width: 0;
        }

        .sb-editor-sticky {
            position: sticky;
            top: 16px;
        }

        .sb-editor-topline {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 16px;
            margin-bottom: 18px;
        }

        .sb-editor-topline-note {
            margin: 0;
            color: #6b7280;
            font-size: 14px;
            max-width: 860px;
            line-height: 1.5;
        }

        .sb-editor-topline-actions {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            justify-content: flex-end;
        }

        .sb-editor-section-head {
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
            margin-bottom: 14px;
        }

        .sb-editor-section-head .sb-panel-title {
            margin: 0;
        }

        .sb-editor-create {
            padding-bottom: 14px;
            margin-bottom: 14px;
            border-bottom: 1px solid #eef2f7;
        }

        .sb-editor-pages {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .sb-editor-page-item {
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            background: #fafafa;
            padding: 12px;
            cursor: pointer;
            transition: border-color .15s ease, background .15s ease, box-shadow .15s ease;
        }

        .sb-editor-page-item:hover {
            border-color: #c7d2fe;
            background: #fcfcff;
        }

        .sb-editor-page-item.is-active {
            border-color: #2563eb;
            background: #eff6ff;
            box-shadow: 0 6px 18px rgba(37, 99, 235, 0.12);
        }

        .sb-editor-page-top {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 12px;
        }

        .sb-editor-page-title {
            margin: 0 0 6px;
            font-size: 15px;
            font-weight: 700;
            color: #111827;
            line-height: 1.2;
        }

        .sb-editor-page-meta {
            display: flex;
            flex-wrap: wrap;
            gap: 6px;
            margin-top: 8px;
        }

        .sb-editor-chip {
            display: inline-flex;
            align-items: center;
            min-height: 24px;
            padding: 0 8px;
            border-radius: 999px;
            background: #f3f4f6;
            color: #4b5563;
            font-size: 12px;
            font-weight: 600;
        }

        .sb-editor-chip--blue {
            background: #eef2ff;
            color: #3730a3;
        }

        .sb-editor-chip--green {
            background: #dcfce7;
            color: #166534;
        }

        .sb-editor-chip--yellow {
            background: #fef3c7;
            color: #92400e;
        }

        .sb-editor-page-actions,
        .sb-editor-block-actions,
        .sb-editor-inspector-actions {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            margin-top: 12px;
        }

        .sb-editor-page-actions .sb-btn,
        .sb-editor-block-actions .sb-btn,
        .sb-editor-inspector-actions .sb-btn {
            height: 32px;
            padding: 0 10px;
            font-size: 12px;
        }

        .sb-editor-canvas {
            background: #ffffff;
            border: 1px solid #e5e7eb;
            border-radius: 18px;
            box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
            overflow: hidden;
        }

        .sb-editor-canvas-head {
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
            padding: 14px 18px;
            border-bottom: 1px solid #e5e7eb;
            background: #f9fafb;
        }

        .sb-editor-canvas-title {
            margin: 0;
            font-size: 18px;
            font-weight: 700;
            color: #111827;
        }

        .sb-editor-canvas-sub {
            margin: 4px 0 0;
            font-size: 13px;
            color: #6b7280;
        }

        .sb-editor-canvas-body {
            background: #f8fafc;
            padding: 24px;
            min-height: 720px;
        }

        .sb-editor-page {
            max-width: 980px;
            margin: 0 auto;
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 20px;
            min-height: 620px;
            padding: 24px;
            box-shadow: 0 12px 28px rgba(15, 23, 42, 0.06);
        }

        .sb-editor-page-heading {
            margin: 0 0 18px;
            font-size: 30px;
            line-height: 1.15;
            font-weight: 700;
            color: #111827;
        }

        .sb-editor-addbar {
            display: grid;
            grid-template-columns: repeat(4, minmax(0, 1fr));
            gap: 10px;
            margin-bottom: 18px;
        }

        .sb-editor-add-card {
            border: 1px solid #dbe3f0;
            border-radius: 14px;
            background: #fff;
            padding: 12px;
            text-align: left;
            cursor: pointer;
            transition: border-color .15s ease, transform .15s ease, box-shadow .15s ease;
        }

        .sb-editor-add-card:hover {
            border-color: #93c5fd;
            transform: translateY(-1px);
            box-shadow: 0 8px 18px rgba(37, 99, 235, 0.08);
        }

        .sb-editor-add-card__title {
            display: block;
            font-size: 14px;
            font-weight: 700;
            color: #111827;
            margin-bottom: 4px;
        }

        .sb-editor-add-card__text {
            display: block;
            font-size: 12px;
            color: #6b7280;
            line-height: 1.4;
        }

        .sb-editor-blocks {
            display: flex;
            flex-direction: column;
            gap: 14px;
        }

        .sb-editor-block {
            border: 1px solid #e5e7eb;
            border-radius: 16px;
            background: #fff;
            padding: 14px;
            transition: border-color .15s ease, box-shadow .15s ease, background .15s ease;
            cursor: pointer;
        }

        .sb-editor-block:hover {
            border-color: #c7d2fe;
            box-shadow: 0 8px 20px rgba(37, 99, 235, 0.08);
        }

        .sb-editor-block.is-active {
            border-color: #2563eb;
            background: #f8fbff;
            box-shadow: 0 10px 24px rgba(37, 99, 235, 0.12);
        }

        .sb-editor-block-head {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 12px;
            margin-bottom: 10px;
        }

        .sb-editor-block-title {
            margin: 0;
            font-size: 14px;
            font-weight: 700;
            color: #111827;
        }

        .sb-editor-block-preview {
            border: 1px solid #eef2f7;
            background: #fff;
            border-radius: 12px;
            padding: 12px;
            font-size: 14px;
            line-height: 1.6;
            color: #374151;
            min-height: 52px;
        }

        .sb-editor-empty-big {
            padding: 30px 20px;
            text-align: center;
            color: #6b7280;
            border: 1px dashed #d1d5db;
            border-radius: 16px;
            background: #fff;
        }

        .sb-editor-empty-big strong {
            display: block;
            color: #111827;
            margin-bottom: 6px;
            font-size: 16px;
        }

        .sb-editor-note {
            font-size: 13px;
            color: #6b7280;
            line-height: 1.5;
            margin-top: -2px;
            margin-bottom: 12px;
        }

        .sb-editor-json-actions {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
            margin-top: 12px;
        }

        @media (max-width: 1440px) {
            .sb-editor-shell {
                grid-template-columns: 300px minmax(0, 1fr) 330px;
            }

            .sb-editor-addbar {
                grid-template-columns: repeat(2, minmax(0, 1fr));
            }
        }

        @media (max-width: 1180px) {
            .sb-editor-shell {
                grid-template-columns: 320px minmax(0, 1fr);
            }

            .sb-editor-col--right {
                grid-column: 1 / -1;
            }

            .sb-editor-sticky {
                position: static;
            }
        }

        @media (max-width: 900px) {
            .sb-editor-topline {
                flex-direction: column;
                align-items: stretch;
            }

            .sb-editor-topline-actions {
                justify-content: flex-start;
            }

            .sb-editor-shell {
                grid-template-columns: 1fr;
            }

            .sb-editor-addbar {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body class="sb-admin-body">
<div class="sb-page">
    <div class="sb-topbar">
        <div class="sb-topbar-left">
            <a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← К списку сайтов</a>
            <h1 class="sb-title">Редактор сайта</h1>
            <p class="sb-subtitle">siteId = <?= (int)$siteId ?></p>
        </div>
    </div>

    <div class="sb-editor-topline">
        <p class="sb-editor-topline-note">
            Слева — структура страниц. По центру — полотно текущей страницы. Справа — свойства выбранной страницы или блока.
        </p>
        <div class="sb-editor-topline-actions">
            <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/public.php?siteId=<?= (int)$siteId ?>" target="_blank">Открыть публичную</a>
            <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/layout.php?siteId=<?= (int)$siteId ?>">Layout</a>
            <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
        </div>
    </div>

    <div class="sb-editor-shell">
        <div class="sb-editor-col">
            <div class="sb-editor-sticky">
                <div class="sb-panel">
                    <div class="sb-editor-section-head">
                        <h2 class="sb-panel-title">Страницы</h2>
                        <span class="sb-badge">siteId <?= (int)$siteId ?></span>
                    </div>

                    <div class="sb-editor-create">
                        <div class="sb-form-row align-end">
                            <div class="sb-field">
                                <label for="newPageTitle">Название страницы</label>
                                <input class="sb-input" type="text" id="newPageTitle" placeholder="Например: Главная">
                            </div>
                        </div>

                        <div class="sb-form-row align-end" style="margin-top:12px;">
                            <div class="sb-field">
                                <label for="newPageSlug">Slug</label>
                                <input class="sb-input" type="text" id="newPageSlug" placeholder="Например: home">
                            </div>

                            <div class="sb-field">
                                <label for="newPageParentId">Родитель</label>
                                <select class="sb-select" id="newPageParentId">
                                    <option value="0">Без родителя</option>
                                </select>
                            </div>
                        </div>

                        <div class="sb-form-row" style="margin-top:12px;">
                            <button class="sb-btn sb-btn-primary" type="button" id="createPageBtn">Создать страницу</button>
                        </div>
                    </div>

                    <div id="pagesList" class="sb-editor-pages">
                        <div class="sb-empty">Загрузка страниц...</div>
                    </div>
                </div>
            </div>
        </div>

        <div class="sb-editor-col">
            <div class="sb-editor-canvas">
                <div class="sb-editor-canvas-head">
                    <div>
                        <h2 class="sb-editor-canvas-title" id="canvasPageTitle">Страница</h2>
                        <p class="sb-editor-canvas-sub" id="canvasPageMeta">Выберите страницу слева</p>
                    </div>
                    <div class="sb-toolbar">
                        <button class="sb-btn sb-btn-light sb-btn-small" type="button" id="movePageUpBtn">Страницу ↑</button>
                        <button class="sb-btn sb-btn-light sb-btn-small" type="button" id="movePageDownBtn">Страницу ↓</button>
                        <button class="sb-btn sb-btn-primary sb-btn-small" type="button" id="publishPageBtn">Опубликовать</button>
                    </div>
                </div>

                <div class="sb-editor-canvas-body">
                    <div class="sb-editor-page">
                        <h2 class="sb-editor-page-heading" id="pagePreviewHeading">Выберите страницу</h2>

                        <div class="sb-editor-addbar">
                            <button class="sb-editor-add-card" type="button" data-add-block="heading">
                                <span class="sb-editor-add-card__title">Заголовок</span>
                                <span class="sb-editor-add-card__text">Большой заголовок или подзаголовок секции</span>
                            </button>
                            <button class="sb-editor-add-card" type="button" data-add-block="text">
                                <span class="sb-editor-add-card__title">Текст</span>
                                <span class="sb-editor-add-card__text">Абзацы, списки и обычный контент</span>
                            </button>
                            <button class="sb-editor-add-card" type="button" data-add-block="button">
                                <span class="sb-editor-add-card__title">Кнопка</span>
                                <span class="sb-editor-add-card__text">CTA-кнопка со ссылкой</span>
                            </button>
                            <button class="sb-editor-add-card" type="button" data-add-block="html">
                                <span class="sb-editor-add-card__title">HTML</span>
                                <span class="sb-editor-add-card__text">Произвольный HTML-блок</span>
                            </button>
                        </div>

                        <div id="blocksList" class="sb-editor-blocks">
                            <div class="sb-editor-empty-big">
                                <strong>Страница не выбрана</strong>
                                Выбери страницу слева, чтобы редактировать ее блоки
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div class="sb-editor-col sb-editor-col--right">
            <div class="sb-editor-sticky">
                <div class="sb-panel">
                    <h2 class="sb-panel-title">Свойства страницы</h2>
                    <p class="sb-editor-note">Здесь меняются заголовок, slug и статус текущей страницы.</p>

                    <div class="sb-field">
                        <label for="pageTitleInput">Название</label>
                        <input class="sb-input" type="text" id="pageTitleInput">
                    </div>

                    <div class="sb-field" style="margin-top:12px;">
                        <label for="pageSlugInput">Slug</label>
                        <input class="sb-input" type="text" id="pageSlugInput">
                    </div>

                    <div class="sb-field" style="margin-top:12px;">
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

                <div class="sb-panel">
                    <h2 class="sb-panel-title">Свойства блока</h2>
                    <div id="blockInspectorEmpty" class="sb-empty">
                        Выбери блок в центре страницы
                    </div>

                    <div id="blockInspector" class="sb-hidden">
                        <div class="sb-field">
                            <label for="blockTypeInput">Тип</label>
                            <input class="sb-input" type="text" id="blockTypeInput" disabled>
                        </div>

                        <div class="sb-field" style="margin-top:12px;">
                            <label for="blockContentInput">Контент (JSON)</label>
                            <textarea class="sb-textarea" id="blockContentInput"></textarea>
                        </div>

                        <div class="sb-field" style="margin-top:12px;">
                            <label for="blockPropsInput">Свойства (JSON)</label>
                            <textarea class="sb-textarea" id="blockPropsInput"></textarea>
                        </div>

                        <div class="sb-editor-json-actions">
                            <button class="sb-btn sb-btn-primary" type="button" id="saveBlockBtn">Сохранить блок</button>
                            <button class="sb-btn sb-btn-light" type="button" id="duplicateBlockBtn">Дублировать</button>
                            <button class="sb-btn sb-btn-light" type="button" id="moveBlockUpBtn">Блок ↑</button>
                            <button class="sb-btn sb-btn-light" type="button" id="moveBlockDownBtn">Блок ↓</button>
                            <button class="sb-btn sb-btn-danger" type="button" id="deleteBlockBtn">Удалить</button>
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
    var newPageParentId = document.getElementById('newPageParentId');

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

    function pageHasChildren(pageId) {
        return state.pages.some(function (page) {
            return Number(page.parentId || 0) === Number(pageId || 0);
        });
    }

    function buildPageTree(pages, parentId, depth, result) {
        result = result || [];
        depth = depth || 0;

        var branch = pages
            .filter(function (page) {
                return Number(page.parentId || 0) === Number(parentId || 0);
            })
            .sort(function (a, b) {
                var sortCmp = Number(a.sort || 0) - Number(b.sort || 0);
                if (sortCmp !== 0) return sortCmp;
                return Number(a.id || 0) - Number(b.id || 0);
            });

        branch.forEach(function (page) {
            result.push({
                page: page,
                depth: depth
            });
            buildPageTree(pages, Number(page.id || 0), depth + 1, result);
        });

        return result;
    }

    async function loadSite() {
        var res = await api('site.get', {siteId: siteId});
        state.site = res.site || null;
    }

    async function loadPages() {
        var res = await api('page.list', {siteId: siteId});
        state.pages = Array.isArray(res.pages) ? res.pages : [];

        if (!state.currentPageId && state.pages.length) {
            state.currentPageId = Number(state.pages[0].id || 0);
        }

        fillParentOptions();
        renderPages();
        fillPageForm();
        updateCanvasHeader();
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
        updateCanvasHeader();
    }

    function fillParentOptions() {
        var currentValue = String(newPageParentId.value || '0');
        var html = '<option value="0">Без родителя</option>';

        state.pages.forEach(function (page) {
            html += '<option value="' + Number(page.id || 0) + '">' + escapeHtml(page.title || ('Страница #' + page.id)) + '</option>';
        });

        newPageParentId.innerHTML = html;
        newPageParentId.value = currentValue;
    }

    function renderPages() {
        if (!state.pages.length) {
            pagesList.innerHTML = '<div class="sb-empty">Страниц пока нет</div>';
            return;
        }

        var tree = buildPageTree(state.pages, 0, 0, []);

        pagesList.innerHTML = tree.map(function (item) {
            var page = item.page;
            var depth = item.depth;
            var active = Number(page.id || 0) === state.currentPageId ? ' is-active' : '';
            var hasChildren = pageHasChildren(page.id);
            var status = String(page.status || 'draft');

            return ''
                + '<div class="sb-editor-page-item' + active + '" data-page-id="' + Number(page.id || 0) + '" style="margin-left:' + (depth * 18) + 'px;">'
                + '  <div class="sb-editor-page-top">'
                + '      <div>'
                + '          <h3 class="sb-editor-page-title">' + escapeHtml(page.title || '') + '</h3>'
                + '          <div class="sb-editor-page-meta">'
                +               '<span class="sb-editor-chip">' + escapeHtml(page.slug || '') + '</span>'
                +               '<span class="sb-editor-chip ' + (status === 'published' ? 'sb-editor-chip--green' : 'sb-editor-chip--yellow') + '">' + escapeHtml(status) + '</span>'
                +               (hasChildren ? '<span class="sb-editor-chip sb-editor-chip--blue">section</span>' : '')
                + '          </div>'
                + '      </div>'
                + '  </div>'
                + '  <div class="sb-editor-page-actions">'
                + '      <button class="sb-btn sb-btn-light" type="button" data-select-page="' + Number(page.id || 0) + '">Открыть</button>'
                + '  </div>'
                + '</div>';
        }).join('');
    }

    function updateCanvasHeader() {
        var page = getCurrentPage();
        var pageTitle = document.getElementById('canvasPageTitle');
        var pageMeta = document.getElementById('canvasPageMeta');
        var previewHeading = document.getElementById('pagePreviewHeading');

        if (!page) {
            pageTitle.textContent = 'Страница';
            pageMeta.textContent = 'Выберите страницу слева';
            previewHeading.textContent = 'Выберите страницу';
            return;
        }

        pageTitle.textContent = page.title || 'Страница';
        pageMeta.textContent = 'slug: ' + (page.slug || '') + ' · статус: ' + (page.status || 'draft') + ' · блоков: ' + state.blocks.length;
        previewHeading.textContent = page.title || 'Страница';
    }

    function blockPreviewText(block) {
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
            return (content.html || '').slice(0, 220) || '[пустой HTML]';
        }

        try {
            return JSON.stringify(content);
        } catch (e) {
            return '[контент блока]';
        }
    }

    function renderBlocks() {
        if (!state.currentPageId) {
            blocksList.innerHTML = ''
                + '<div class="sb-editor-empty-big">'
                + '   <strong>Страница не выбрана</strong>'
                + '   Выбери страницу слева, чтобы редактировать блоки'
                + '</div>';
            return;
        }

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
                + '  <div class="sb-editor-block-actions">'
                + '      <button class="sb-btn sb-btn-light" type="button" data-select-block="' + Number(block.id || 0) + '">Выбрать</button>'
                + '  </div>'
                + '</div>';
        }).join('');
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
        var slug = (document.getElementById('newPageSlug').value || '').trim();
        var parentId = Number(newPageParentId.value || 0);

        if (!title) {
            alert('Введите название страницы');
            return;
        }

        await api('page.create', {
            siteId: siteId,
            title: title,
            slug: slug
        });

        document.getElementById('newPageTitle').value = '';
        document.getElementById('newPageSlug').value = '';

        await loadPages();

        if (parentId > 0) {
            var lastPage = state.pages[state.pages.length - 1];
            if (lastPage && Number(lastPage.id || 0) > 0) {
                await api('page.setParent', {
                    id: Number(lastPage.id || 0),
                    parentId: parentId
                });
                await loadPages();
            }
        }

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
