Вот улучшенный index.php целиком. Я сделал его более собранным: компактнее форма создания, сильнее карточки сайтов, лучше сгруппированы действия, меньше ощущения “простыни”.

Полный файл /local/sitebuilder/index.php

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
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder</title>
    <?php $APPLICATION->ShowHead(); ?>
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    <style>
        .sb-index-hero {
            display: grid;
            grid-template-columns: minmax(320px, 760px) 1fr;
            gap: 20px;
            align-items: start;
            margin-bottom: 20px;
        }

        .sb-index-create-card {
            background: linear-gradient(180deg, #ffffff 0%, #f8fbff 100%);
        }

        .sb-index-create-inner {
            max-width: 760px;
        }

        .sb-index-create-row {
            display: grid;
            grid-template-columns: minmax(220px, 1.2fr) minmax(220px, 1fr) auto;
            gap: 12px;
            align-items: end;
        }

        .sb-index-stats {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 12px;
        }

        .sb-index-stat {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 16px;
            box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
        }

        .sb-index-stat-label {
            font-size: 13px;
            color: #6b7280;
            margin-bottom: 8px;
        }

        .sb-index-stat-value {
            font-size: 28px;
            line-height: 1;
            font-weight: 700;
            color: #111827;
        }

        .sb-index-section-head {
            display: flex;
            justify-content: space-between;
            align-items: center;
            gap: 12px;
            margin-bottom: 14px;
        }

        .sb-index-section-head .sb-panel-title {
            margin: 0;
        }

        .sb-sites-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(360px, 1fr));
            gap: 16px;
        }

        .sb-site-card {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 16px;
            padding: 18px;
            display: flex;
            flex-direction: column;
            gap: 16px;
            box-shadow: 0 2px 10px rgba(15, 23, 42, 0.04);
        }

        .sb-site-head {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 12px;
        }

        .sb-site-head-main {
            min-width: 0;
        }

        .sb-site-name {
            margin: 0 0 6px;
            font-size: 20px;
            font-weight: 700;
            line-height: 1.2;
            word-break: break-word;
            color: #111827;
        }

        .sb-site-slug {
            display: inline-flex;
            align-items: center;
            min-height: 28px;
            padding: 0 10px;
            border-radius: 999px;
            background: #f3f4f6;
            color: #4b5563;
            font-size: 13px;
            font-weight: 600;
            max-width: 100%;
            word-break: break-word;
        }

        .sb-site-meta-grid {
            display: grid;
            grid-template-columns: repeat(2, 1fr);
            gap: 10px 14px;
            padding: 14px;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            background: #fafafa;
        }

        .sb-site-meta-item {
            min-width: 0;
        }

        .sb-site-meta-label {
            font-size: 12px;
            color: #6b7280;
            margin-bottom: 4px;
        }

        .sb-site-meta-value {
            font-size: 14px;
            color: #111827;
            word-break: break-word;
        }

        .sb-site-actions {
            display: flex;
            flex-direction: column;
            gap: 10px;
        }

        .sb-site-actions-row {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
        }

        .sb-site-actions-row a,
        .sb-site-actions-row button {
            text-decoration: none;
        }

        .sb-site-actions-row--secondary .sb-btn {
            background: #f8fafc;
            color: #334155;
        }

        .sb-site-actions-row--secondary .sb-btn:hover {
            background: #eef2f7;
        }

        .sb-site-actions-row--danger {
            justify-content: flex-end;
        }

        .sb-site-actions-row--danger .sb-btn {
            min-width: 110px;
        }

        .sb-site-created {
            font-size: 13px;
            color: #6b7280;
            margin-top: -4px;
        }

        @media (max-width: 1180px) {
            .sb-index-hero {
                grid-template-columns: 1fr;
            }
        }

        @media (max-width: 860px) {
            .sb-index-create-row {
                grid-template-columns: 1fr;
            }

            .sb-index-stats {
                grid-template-columns: 1fr 1fr;
            }
        }

        @media (max-width: 640px) {
            .sb-index-stats,
            .sb-site-meta-grid {
                grid-template-columns: 1fr;
            }

            .sb-site-actions-row--danger {
                justify-content: flex-start;
            }
        }
    </style>
</head>
<body class="sb-admin-body">
<div class="sb-page">
    <div class="sb-topbar">
        <div>
            <h1 class="sb-title">SiteBuilder</h1>
            <p class="sb-subtitle">Управление сайтами конструктора</p>
        </div>
        <div class="sb-userbox">
            Пользователь:
            <strong><?= htmlspecialchars((string)$USER->GetLogin(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></strong>
            (ID <?= (int)$USER->GetID() ?>)
        </div>
    </div>

    <div class="sb-index-hero">
        <div class="sb-panel sb-index-create-card">
            <div class="sb-index-create-inner">
                <h2 class="sb-panel-title">Создать сайт</h2>
                <div class="sb-index-create-row">
                    <div class="sb-field">
                        <label for="siteName">Название сайта</label>
                        <input class="sb-input" type="text" id="siteName" placeholder="Например: Корпоративный портал">
                    </div>
                    <div class="sb-field">
                        <label for="siteSlug">Slug</label>
                        <input class="sb-input" type="text" id="siteSlug" placeholder="Например: corp-portal">
                    </div>
                    <button type="button" class="sb-btn sb-btn-primary" id="createSiteBtn">Создать</button>
                </div>
            </div>
        </div>

        <div class="sb-index-stats">
            <div class="sb-index-stat">
                <div class="sb-index-stat-label">Всего сайтов</div>
                <div class="sb-index-stat-value" id="statSitesCount">0</div>
            </div>
            <div class="sb-index-stat">
                <div class="sb-index-stat-label">Доступно текущему пользователю</div>
                <div class="sb-index-stat-value" id="statAccessibleCount">0</div>
            </div>
        </div>
    </div>

    <div class="sb-panel">
        <div class="sb-index-section-head">
            <h2 class="sb-panel-title">Сайты</h2>
            <button type="button" class="sb-btn sb-btn-light sb-btn-small" id="reloadBtn">Обновить список</button>
        </div>
        <div id="sitesContainer">
            <div class="sb-empty">Загрузка списка сайтов...</div>
        </div>
    </div>

    <div class="sb-panel">
        <h2 class="sb-panel-title">Отладка</h2>
        <div id="output" class="sb-output">Здесь будут ответы API...</div>
    </div>
</div>

<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var output = document.getElementById('output');
    var sitesContainer = document.getElementById('sitesContainer');
    var statSitesCount = document.getElementById('statSitesCount');
    var statAccessibleCount = document.getElementById('statAccessibleCount');

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

    function updateStats(sites) {
        var count = Array.isArray(sites) ? sites.length : 0;
        statSitesCount.textContent = String(count);
        statAccessibleCount.textContent = String(count);
    }

    function siteCard(site) {
        var id = Number(site.id || 0);
        var name = escapeHtml(site.name || '');
        var slug = escapeHtml(site.slug || '');
        var createdAt = escapeHtml(site.createdAt || '');
        var homePageId = Number(site.homePageId || 0);
        var diskFolderId = Number(site.diskFolderId || 0);

        return ''
            + '<div class="sb-site-card">'
            + '  <div class="sb-site-head">'
            + '    <div class="sb-site-head-main">'
            + '      <h3 class="sb-site-name">' + name + '</h3>'
            + '      <div class="sb-site-slug">' + slug + '</div>'
            + '    </div>'
            + '    <span class="sb-badge">site #' + id + '</span>'
            + '  </div>'
            + ''
            + '  <div class="sb-site-meta-grid">'
            + '    <div class="sb-site-meta-item">'
            + '      <div class="sb-site-meta-label">ID</div>'
            + '      <div class="sb-site-meta-value">' + id + '</div>'
            + '    </div>'
            + '    <div class="sb-site-meta-item">'
            + '      <div class="sb-site-meta-label">Home page ID</div>'
            + '      <div class="sb-site-meta-value">' + homePageId + '</div>'
            + '    </div>'
            + '    <div class="sb-site-meta-item">'
            + '      <div class="sb-site-meta-label">Disk folder ID</div>'
            + '      <div class="sb-site-meta-value">' + diskFolderId + '</div>'
            + '    </div>'
            + '    <div class="sb-site-meta-item">'
            + '      <div class="sb-site-meta-label">Создан</div>'
            + '      <div class="sb-site-meta-value">' + createdAt + '</div>'
            + '    </div>'
            + '  </div>'
            + ''
            + '  <div class="sb-site-actions">'
            + '    <div class="sb-site-actions-row">'
            + '      <a class="sb-btn sb-btn-primary sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + id + '">Редактор</a>'
            + '      <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/public.php?siteId=' + id + '" target="_blank">Публичная</a>'
            + '    </div>'
            + '    <div class="sb-site-actions-row sb-site-actions-row--secondary">'
            + '      <a class="sb-btn sb-btn-small" href="' + BASE_PATH + '/layout.php?siteId=' + id + '">Layout</a>'
            + '      <a class="sb-btn sb-btn-small" href="' + BASE_PATH + '/menu.php?siteId=' + id + '">Меню</a>'
            + '      <a class="sb-btn sb-btn-small" href="' + BASE_PATH + '/files.php?siteId=' + id + '">Файлы</a>'
            + '      <a class="sb-btn sb-btn-small" href="' + BASE_PATH + '/settings.php?siteId=' + id + '">Настройки</a>'
            + '    </div>'
            + '    <div class="sb-site-actions-row sb-site-actions-row--danger">'
            + '      <button type="button" class="sb-btn sb-btn-danger sb-btn-small js-delete-site" data-id="' + id + '">Удалить</button>'
            + '    </div>'
            + '  </div>'
            + '</div>';
    }

    function renderSites(sites) {
        updateStats(sites);

        if (!Array.isArray(sites) || !sites.length) {
            sitesContainer.innerHTML = '<div class="sb-empty">Сайтов пока нет</div>';
            return;
        }

        var html = '<div class="sb-sites-grid">';
        for (var i = 0; i < sites.length; i++) {
            html += siteCard(sites[i]);
        }
        html += '</div>';

        sitesContainer.innerHTML = html;
    }

    function loadSites() {
        sitesContainer.innerHTML = '<div class="sb-empty">Загрузка...</div>';

        api('site.list', {}, function (res) {
            if (!res || res.ok !== true) {
                updateStats([]);
                sitesContainer.innerHTML = '<div class="sb-empty">Не удалось загрузить список сайтов</div>';
                return;
            }
            renderSites(res.sites || []);
        });
    }

    function createSite() {
        var nameInput = document.getElementById('siteName');
        var slugInput = document.getElementById('siteSlug');

        var name = (nameInput.value || '').trim();
        var slug = (slugInput.value || '').trim();

        if (!name) {
            alert('Введите название сайта');
            nameInput.focus();
            return;
        }

        api('site.create', {
            name: name,
            slug: slug
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось создать сайт');
                return;
            }

            nameInput.value = '';
            slugInput.value = '';
            loadSites();
        });
    }

    function deleteSite(id) {
        if (!id) {
            return;
        }

        if (!confirm('Удалить сайт #' + id + '?')) {
            return;
        }

        api('site.delete', {
            id: id
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось удалить сайт');
                return;
            }

            loadSites();
        });
    }

    document.getElementById('createSiteBtn').addEventListener('click', createSite);
    document.getElementById('reloadBtn').addEventListener('click', loadSites);

    document.addEventListener('click', function (e) {
        var btn = e.target.closest('.js-delete-site');
        if (!btn) {
            return;
        }

        var id = parseInt(btn.getAttribute('data-id'), 10) || 0;
        deleteSite(id);
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

    loadSites();
})();
</script>
</body>
</html>

Это уже выглядит заметно лучше как стартовая страница конструктора.

Дальше логично в таком же ключе улучшить editor.php, уже не просто “под admin.css”, а именно визуально собрать его в более сильный рабочий экран.