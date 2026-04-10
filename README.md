Вот полностью готовый index.php под admin.css.

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
        .sb-sites-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(340px, 1fr));
            gap: 16px;
        }

        .sb-site-card {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 16px;
            display: flex;
            flex-direction: column;
            gap: 12px;
        }

        .sb-site-head {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 12px;
        }

        .sb-site-name {
            margin: 0;
            font-size: 18px;
            font-weight: 700;
            word-break: break-word;
        }

        .sb-site-actions {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
        }

        .sb-site-actions a,
        .sb-site-actions button {
            text-decoration: none;
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

    <div class="sb-panel">
        <h2 class="sb-panel-title">Создать сайт</h2>
        <div class="sb-form-row align-end">
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

    <div class="sb-panel">
        <div style="display:flex; justify-content:space-between; align-items:center; gap:12px; margin-bottom:14px;">
            <h2 class="sb-panel-title" style="margin:0;">Сайты</h2>
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
            + '    <div>'
            + '      <h3 class="sb-site-name">' + name + '</h3>'
            + '      <div class="sb-meta">'
            + '        <div><strong>ID:</strong> ' + id + '</div>'
            + '        <div><strong>Slug:</strong> ' + slug + '</div>'
            + '        <div><strong>Home page ID:</strong> ' + homePageId + '</div>'
            + '        <div><strong>Disk folder ID:</strong> ' + diskFolderId + '</div>'
            + '        <div><strong>Создан:</strong> ' + createdAt + '</div>'
            + '      </div>'
            + '    </div>'
            + '    <span class="sb-badge">site #' + id + '</span>'
            + '  </div>'
            + ''
            + '  <div class="sb-site-actions">'
            + '    <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + id + '">Редактор</a>'
            + '    <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/layout.php?siteId=' + id + '">Layout</a>'
            + '    <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/menu.php?siteId=' + id + '">Меню</a>'
            + '    <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/files.php?siteId=' + id + '">Файлы</a>'
            + '    <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/settings.php?siteId=' + id + '">Настройки</a>'
            + '    <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/public.php?siteId=' + id + '" target="_blank">Публичная</a>'
            + '    <button type="button" class="sb-btn sb-btn-danger sb-btn-small js-delete-site" data-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }

    function renderSites(sites) {
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

Следующим таким же пакетом логично прислать menu.php, потому что он тоже часто открывается.