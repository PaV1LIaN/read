Вот полный index.php:

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
$canCreateSite = $USER->IsAdmin();
$isBitrixAdmin = $USER->IsAdmin();
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder</title>
    <?php $APPLICATION->ShowHead(); ?>
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/dashboard.css">

    <style>
        .sb-role-badge {
            display: inline-flex;
            align-items: center;
            min-height: 24px;
            padding: 0 8px;
            border-radius: 999px;
            font-size: 12px;
            font-weight: 700;
            white-space: nowrap;
        }

        .sb-role-badge--owner {
            background: #dcfce7;
            color: #166534;
        }

        .sb-role-badge--admin {
            background: #e0f2fe;
            color: #075985;
        }

        .sb-role-badge--editor {
            background: #eef2ff;
            color: #3730a3;
        }

        .sb-role-badge--viewer {
            background: #f3f4f6;
            color: #374151;
        }

        .sb-group-cell {
            display: flex;
            flex-direction: column;
            gap: 6px;
            min-width: 150px;
        }

        .sb-group-cell__id {
            font-size: 12px;
            color: #6b7280;
        }

        .sb-actions-stack {
            display: flex;
            flex-wrap: wrap;
            gap: 6px;
            min-width: 180px;
        }

        .sb-actions-stack .sb-btn {
            height: 30px;
            padding: 0 9px;
            font-size: 12px;
        }

        .sb-muted {
            color: #9ca3af;
            font-size: 12px;
        }

        .sb-user-sites-table .sb-site-cell__title {
            font-size: 15px;
            font-weight: 700;
        }

        .sb-user-sites-table .sb-actions-stack {
            justify-content: flex-end;
        }

        .sb-modal[hidden] {
            display: none !important;
        }

        .sb-modal {
            position: fixed;
            inset: 0;
            z-index: 10000;
            display: flex;
            align-items: center;
            justify-content: center;
            padding: 24px;
        }

        .sb-modal__backdrop {
            position: absolute;
            inset: 0;
            background: rgba(15, 23, 42, 0.45);
            backdrop-filter: blur(4px);
        }

        .sb-modal__dialog {
            position: relative;
            width: min(520px, 100%);
            background: #fff;
            border-radius: 20px;
            box-shadow: 0 24px 70px rgba(15, 23, 42, 0.28);
            overflow: hidden;
            border: 1px solid #e5e7eb;
        }

        .sb-modal__head {
            display: flex;
            justify-content: space-between;
            gap: 16px;
            padding: 20px 22px;
            border-bottom: 1px solid #eef2f7;
            background: #f9fafb;
        }

        .sb-modal__title {
            margin: 0;
            font-size: 20px;
            font-weight: 800;
            color: #111827;
        }

        .sb-modal__subtitle {
            margin: 6px 0 0;
            font-size: 13px;
            color: #6b7280;
            line-height: 1.45;
        }

        .sb-modal__close {
            width: 34px;
            height: 34px;
            border: 1px solid #e5e7eb;
            border-radius: 10px;
            background: #fff;
            cursor: pointer;
            font-size: 22px;
            line-height: 1;
            color: #6b7280;
        }

        .sb-modal__close:hover {
            background: #f3f4f6;
            color: #111827;
        }

        .sb-modal__body {
            padding: 22px;
        }

        .sb-modal__footer {
            display: flex;
            justify-content: flex-end;
            gap: 10px;
            padding: 16px 22px;
            border-top: 1px solid #eef2f7;
            background: #f9fafb;
        }
    </style>
</head>
<body class="sb-admin-body">
<div class="sb-page">
    <div class="sb-dashboard">
        <header class="sb-dashboard-topbar">
            <div class="sb-dashboard-topbar__main">
                <h1 class="sb-dashboard-title">SiteBuilder</h1>
                <p class="sb-dashboard-subtitle">
                    Управление сайтами конструктора
                </p>
            </div>

            <div class="sb-dashboard-actions">
                <input class="sb-dashboard-search" type="text" id="dashboardSearch" placeholder="Поиск по сайтам">
                <button class="sb-btn sb-btn-light" type="button" id="reloadBtn">Обновить</button>

                <?php if ($canCreateSite): ?>
                    <button class="sb-btn sb-btn-primary" type="button" id="createSiteQuickBtn">Создать сайт</button>
                <?php endif; ?>
            </div>
        </header>

        <section class="sb-dash-card">
            <div class="sb-dash-card__head">
                <h2>Список сайтов</h2>
                <div class="sb-dash-toolbar">
                    <span class="sb-badge" id="sitesCountBadge">0 сайтов</span>
                </div>
            </div>

            <div class="sb-dash-table-wrap">
                <table class="sb-dash-table <?= $isBitrixAdmin ? '' : 'sb-user-sites-table' ?>">
                    <thead>
                    <?php if ($isBitrixAdmin): ?>
                        <tr>
                            <th>ID</th>
                            <th>Сайт</th>
                            <th>Роль</th>
                            <th>Slug</th>
                            <th>Домашняя</th>
                            <th>Группа Битрикс24</th>
                            <th>Диск</th>
                            <th>Изменен</th>
                            <th>Действия</th>
                        </tr>
                    <?php else: ?>
                        <tr>
                            <th>Сайт</th>
                            <th style="text-align:right;">Действия</th>
                        </tr>
                    <?php endif; ?>
                    </thead>

                    <tbody id="sitesTableBody">
                    <tr>
                        <td colspan="<?= $isBitrixAdmin ? 9 : 2 ?>">Загрузка...</td>
                    </tr>
                    </tbody>
                </table>
            </div>
        </section>

        <?php if ($canCreateSite): ?>
            <div class="sb-modal" id="createSiteModal" hidden>
                <div class="sb-modal__backdrop" data-close-create-modal></div>

                <div class="sb-modal__dialog">
                    <div class="sb-modal__head">
                        <div>
                            <h2 class="sb-modal__title">Создать сайт</h2>
                            <p class="sb-modal__subtitle">
                                Новый сайт автоматически получит рабочую группу Битрикс24.
                            </p>
                        </div>

                        <button class="sb-modal__close" type="button" data-close-create-modal>×</button>
                    </div>

                    <div class="sb-modal__body">
                        <div class="sb-field">
                            <label for="siteName">Название сайта</label>
                            <input class="sb-input" type="text" id="siteName" placeholder="Например: Корпоративный портал">
                        </div>

                        <div class="sb-field" style="margin-top:12px;">
                            <label for="siteSlug">Slug</label>
                            <input class="sb-input" type="text" id="siteSlug" placeholder="Например: corp-portal">
                        </div>
                    </div>

                    <div class="sb-modal__footer">
                        <button class="sb-btn sb-btn-light" type="button" data-close-create-modal>Отмена</button>
                        <button class="sb-btn sb-btn-primary" type="button" id="createSiteBtn">Создать</button>
                    </div>
                </div>
            </div>
        <?php endif; ?>

        <?php if ($isBitrixAdmin): ?>
            <section class="sb-dash-card">
                <div class="sb-dash-card__head">
                    <h2>Отладка</h2>
                </div>
                <div id="output" class="sb-output">Здесь будут ответы API...</div>
            </section>
        <?php else: ?>
            <div id="output" style="display:none;"></div>
        <?php endif; ?>
    </div>
</div>

<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';
    var CAN_CREATE_SITE = <?= $canCreateSite ? 'true' : 'false' ?>;
    var IS_BITRIX_ADMIN = <?= $isBitrixAdmin ? 'true' : 'false' ?>;

    var output = document.getElementById('output');
    var sitesTableBody = document.getElementById('sitesTableBody');
    var sitesCountBadge = document.getElementById('sitesCountBadge');
    var searchInput = document.getElementById('dashboardSearch');

    var allSites = [];

    function print(data) {
        if (!output) {
            return;
        }

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
        return String(value == null ? '' : value)
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

                if (res && res.ok === true) {
                    if (typeof onSuccess === 'function') {
                        onSuccess(res);
                    }
                    return;
                }

                if (typeof onFailure === 'function') {
                    onFailure(res || {error: 'UNKNOWN'});
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

    function siteStatus(site) {
        return Number(site.homePageId || 0) > 0 ? 'published' : 'draft';
    }

    function yesNoBadge(flag) {
        if (flag) {
            return '<span class="sb-status-badge success">Да</span>';
        }

        return '<span class="sb-status-badge warning">Нет</span>';
    }

    function roleRank(role) {
        role = String(role || '');

        if (role === 'OWNER') return 4;
        if (role === 'ADMIN') return 3;
        if (role === 'EDITOR') return 2;
        if (role === 'VIEWER') return 1;

        return 0;
    }

    function roleBadge(role) {
        role = String(role || 'VIEWER');

        var cls = 'sb-role-badge--viewer';

        if (role === 'OWNER') {
            cls = 'sb-role-badge--owner';
        } else if (role === 'ADMIN') {
            cls = 'sb-role-badge--admin';
        } else if (role === 'EDITOR') {
            cls = 'sb-role-badge--editor';
        }

        return '<span class="sb-role-badge ' + cls + '">' + escapeHtml(role) + '</span>';
    }

    function getBitrixGroupUrl(site) {
        var groupId = Number(site.bitrixGroupId || 0);

        if (site.bitrixGroupUrl) {
            return String(site.bitrixGroupUrl);
        }

        if (groupId > 0) {
            return '/workgroups/group/' + groupId + '/';
        }

        return '';
    }

    function renderGroupCell(site, userRoleRank) {
        var groupId = Number(site.bitrixGroupId || 0);
        var groupUrl = getBitrixGroupUrl(site);

        if (groupId > 0) {
            return ''
                + '<div class="sb-group-cell">'
                + '  <div class="sb-group-cell__id">ID группы: ' + groupId + '</div>'
                + '  <a class="sb-btn sb-btn-light sb-btn-small" href="' + escapeHtml(groupUrl) + '" target="_blank">Открыть группу</a>'
                + '</div>';
        }

        if (userRoleRank >= 4) {
            return ''
                + '<div class="sb-group-cell">'
                + '  <span class="sb-muted">Группа не создана</span>'
                + '  <button class="sb-btn sb-btn-primary sb-btn-small" type="button" data-action="ensure-group" data-site-id="' + Number(site.id || 0) + '">Создать группу</button>'
                + '</div>';
        }

        return '<span class="sb-muted">Нет группы</span>';
    }

    function renderActions(site, userRoleRank) {
        var siteId = Number(site.id || 0);
        var groupId = Number(site.bitrixGroupId || 0);

        var html = '<div class="sb-actions-stack">';

        html += '<a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/public.php?siteId=' + siteId + '" target="_blank">Публичная</a>';

        if (userRoleRank >= 2) {
            html += '<a class="sb-btn sb-btn-primary sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + siteId + '">Редактор</a>';
        }

        if (userRoleRank >= 4 && groupId > 0) {
            html += '<button class="sb-btn sb-btn-light sb-btn-small" type="button" data-action="sync-access" data-site-id="' + siteId + '">Синхр. права</button>';
        }

        if (userRoleRank >= 4) {
            html += '<button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-action="delete-site" data-site-id="' + siteId + '" data-site-name="' + escapeHtml(site.name || '') + '">Удалить</button>';
        }

        html += '</div>';

        return html;
    }

    function renderUserActions(site, userRoleRank) {
        var siteId = Number(site.id || 0);
        var html = '<div class="sb-actions-stack">';

        html += '<a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/public.php?siteId=' + siteId + '" target="_blank">Открыть</a>';

        if (userRoleRank >= 2) {
            html += '<a class="sb-btn sb-btn-primary sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + siteId + '">Редактор</a>';
        }

        html += '</div>';

        return html;
    }

    function renderSitesTable(sites) {
        var colspan = IS_BITRIX_ADMIN ? 9 : 2;

        if (!Array.isArray(sites) || !sites.length) {
            sitesTableBody.innerHTML = '<tr><td colspan="' + colspan + '">Сайтов пока нет</td></tr>';
            sitesCountBadge.textContent = '0 сайтов';
            return;
        }

        sitesCountBadge.textContent = sites.length + ' сайтов';

        var html = '';

        for (var i = 0; i < sites.length; i++) {
            var s = sites[i];
            var siteId = Number(s.id || 0);
            var currentUserRole = String(s.currentUserRole || 'VIEWER');
            var currentUserRoleRank = roleRank(currentUserRole);

            if (!IS_BITRIX_ADMIN) {
                html += ''
                    + '<tr>'
                    + '  <td>'
                    + '    <div class="sb-site-cell">'
                    + '      <div class="sb-site-cell__title">' + escapeHtml(s.name || '') + '</div>'
                    + '    </div>'
                    + '  </td>'
                    + '  <td style="text-align:right;">' + renderUserActions(s, currentUserRoleRank) + '</td>'
                    + '</tr>';

                continue;
            }

            var status = siteStatus(s);
            var statusClass = status === 'published' ? 'success' : 'warning';

            html += ''
                + '<tr>'
                + '  <td>' + siteId + '</td>'
                + '  <td>'
                + '    <div class="sb-site-cell">'
                + '      <div class="sb-site-cell__title">' + escapeHtml(s.name || '') + '</div>'
                + '      <div class="sb-site-cell__meta">created: ' + escapeHtml(s.createdAt || '—') + '</div>'
                + '    </div>'
                + '  </td>'
                + '  <td>' + roleBadge(currentUserRole) + '</td>'
                + '  <td>' + escapeHtml(s.slug || '') + '</td>'
                + '  <td><span class="sb-status-badge ' + statusClass + '">' + escapeHtml(status) + '</span></td>'
                + '  <td>' + renderGroupCell(s, currentUserRoleRank) + '</td>'
                + '  <td>' + yesNoBadge(Number(s.diskFolderId || 0) > 0) + '</td>'
                + '  <td>' + escapeHtml(s.updatedAt || '—') + '</td>'
                + '  <td>' + renderActions(s, currentUserRoleRank) + '</td>'
                + '</tr>';
        }

        sitesTableBody.innerHTML = html;
    }

    function applySearch() {
        var query = String(searchInput.value || '').trim().toLowerCase();

        if (!query) {
            renderSitesTable(allSites);
            return;
        }

        var filtered = allSites.filter(function (site) {
            var haystack = [
                site.name || '',
                site.slug || '',
                site.code || '',
                site.currentUserRole || '',
                String(site.bitrixGroupId || '')
            ].join(' ').toLowerCase();

            return haystack.indexOf(query) !== -1;
        });

        renderSitesTable(filtered);
    }

    function loadSites() {
        api('site.list', {}, function (res) {
            if (!res || res.ok !== true) {
                sitesTableBody.innerHTML = '<tr><td colspan="' + (IS_BITRIX_ADMIN ? 9 : 2) + '">Не удалось загрузить данные</td></tr>';
                sitesCountBadge.textContent = 'Ошибка загрузки';
                return;
            }

            allSites = Array.isArray(res.sites) ? res.sites : [];
            applySearch();
        });
    }

    function openCreateSiteModal() {
        if (!CAN_CREATE_SITE) {
            return;
        }

        var modal = document.getElementById('createSiteModal');
        if (!modal) {
            return;
        }

        modal.hidden = false;

        setTimeout(function () {
            var nameInput = document.getElementById('siteName');
            if (nameInput) {
                nameInput.focus();
            }
        }, 50);
    }

    function closeCreateSiteModal() {
        var modal = document.getElementById('createSiteModal');
        if (!modal) {
            return;
        }

        modal.hidden = true;
    }

    function createSite() {
        if (!CAN_CREATE_SITE) {
            alert('Создавать сайты может только администратор Битрикс24');
            return;
        }

        var nameInput = document.getElementById('siteName');
        var slugInput = document.getElementById('siteSlug');

        if (!nameInput || !slugInput) {
            alert('Форма создания сайта не найдена');
            return;
        }

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
            closeCreateSiteModal();
            loadSites();
        }, function (err) {
            var message = err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR';
            alert('Не удалось создать сайт: ' + message);
        });
    }

    function syncAccess(siteId) {
        api('site.syncAccess', {
            siteId: siteId
        }, function (res) {
            var result = res.result || {};

            alert(
                'Права синхронизированы.\n\n' +
                'Создано: ' + Number(result.created || 0) + '\n' +
                'Обновлено: ' + Number(result.updated || 0) + '\n' +
                'Удалено: ' + Number(result.removed || 0) + '\n' +
                'Без изменений: ' + Number(result.kept || 0)
            );

            loadSites();
        }, function (err) {
            var message = err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR';
            alert('Не удалось синхронизировать права: ' + message);
        });
    }

    function ensureGroup(siteId) {
        api('site.ensureGroup', {
            siteId: siteId
        }, function (res) {
            alert(
                res.created
                    ? 'Группа Битрикс24 создана. ID: ' + Number(res.bitrixGroupId || 0)
                    : 'Группа уже была создана. ID: ' + Number(res.bitrixGroupId || 0)
            );

            loadSites();
        }, function (err) {
            var message = err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR';
            alert('Не удалось создать группу: ' + message);
        });
    }

    function deleteSite(siteId, siteName) {
        var name = siteName || ('siteId ' + siteId);

        var firstConfirm = confirm(
            'Удалить сайт "' + name + '"?\n\n' +
            'Будут удалены страницы, блоки, меню, доступы, шаблоны и layout внутри SiteBuilder.\n' +
            'Группа Битрикс24 и файлы диска сейчас не удаляются автоматически.'
        );

        if (!firstConfirm) {
            return;
        }

        var secondConfirm = confirm(
            'Подтверди удаление ещё раз.\n\n' +
            'Это действие нельзя будет отменить через интерфейс SiteBuilder.'
        );

        if (!secondConfirm) {
            return;
        }

        api('site.delete', {
            id: siteId
        }, function () {
            alert('Сайт удалён');
            loadSites();
        }, function (err) {
            var message = err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR';
            alert('Не удалось удалить сайт: ' + message);
        });
    }

    sitesTableBody.addEventListener('click', function (e) {
        var btn = e.target.closest('[data-action]');
        if (!btn) {
            return;
        }

        var action = btn.getAttribute('data-action');
        var siteId = Number(btn.getAttribute('data-site-id') || 0);

        if (siteId <= 0) {
            return;
        }

        if (action === 'sync-access') {
            syncAccess(siteId);
            return;
        }

        if (action === 'ensure-group') {
            ensureGroup(siteId);
            return;
        }

        if (action === 'delete-site') {
            deleteSite(siteId, btn.getAttribute('data-site-name') || '');
        }
    });

    var createSiteBtn = document.getElementById('createSiteBtn');
    if (createSiteBtn) {
        createSiteBtn.addEventListener('click', createSite);
    }

    var createSiteQuickBtn = document.getElementById('createSiteQuickBtn');
    if (createSiteQuickBtn) {
        createSiteQuickBtn.addEventListener('click', openCreateSiteModal);
    }

    document.querySelectorAll('[data-close-create-modal]').forEach(function (btn) {
        btn.addEventListener('click', closeCreateSiteModal);
    });

    document.addEventListener('keydown', function (e) {
        if (e.key === 'Escape') {
            closeCreateSiteModal();
        }
    });

    document.getElementById('reloadBtn').addEventListener('click', loadSites);
    searchInput.addEventListener('input', applySearch);

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