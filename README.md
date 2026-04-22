Ниже **полностью готовый `index.php` без всей статистики**.

Что я убрал:

* KPI-карточки
* график посещаемости
* популярные сайты
* низкую активность
* последние действия
* сводную активность по системе
* весь фейковый JS:

  * `buildFakeSiteStats`
  * `renderKpis`
  * `renderPopular`
  * `renderLowActivity`
  * `renderLastActions`
  * `renderSystemStats`
  * `buildDashboard`

Что оставил:

* заголовок
* поиск по сайтам
* кнопку создания сайта
* реальный список сайтов через `site.list`
* отладочный блок API
* переход в редактор сайта

Этот файл заменяет текущий `/local/sitebuilder/index.php`. По архиву именно в нем сейчас захардкоженная статистика и фейковые генераторы метрик. 

```php
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
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/dashboard.css">
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
                <button class="sb-btn sb-btn-primary" type="button" id="createSiteQuickBtn">Создать сайт</button>
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
                <table class="sb-dash-table">
                    <thead>
                    <tr>
                        <th>ID</th>
                        <th>Сайт</th>
                        <th>Slug</th>
                        <th>Домашняя страница</th>
                        <th>Меню</th>
                        <th>Диск</th>
                        <th>Создан</th>
                        <th>Изменен</th>
                        <th>Действия</th>
                    </tr>
                    </thead>
                    <tbody id="sitesTableBody">
                    <tr>
                        <td colspan="9">Загрузка...</td>
                    </tr>
                    </tbody>
                </table>
            </div>
        </section>

        <section class="sb-dash-card">
            <div class="sb-dash-card__head">
                <h2>Создать сайт</h2>
            </div>
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
        </section>

        <section class="sb-dash-card">
            <div class="sb-dash-card__head">
                <h2>Отладка</h2>
            </div>
            <div id="output" class="sb-output">Здесь будут ответы API...</div>
        </section>
    </div>
</div>

<script>
(function () {
    var BASE_PATH = '<?= CUtil::JSEscape($basePath) ?>';
    var API_URL = BASE_PATH + '/api.php';

    var output = document.getElementById('output');
    var sitesTableBody = document.getElementById('sitesTableBody');
    var sitesCountBadge = document.getElementById('sitesCountBadge');
    var searchInput = document.getElementById('dashboardSearch');

    var allSites = [];

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
                if (typeof onSuccess === 'function') onSuccess(res);
            },
            onfailure: function (err) {
                print({
                    ok: false,
                    error: 'AJAX_ERROR',
                    detail: err
                });
                if (typeof onFailure === 'function') onFailure(err);
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

    function renderSitesTable(sites) {
        if (!Array.isArray(sites) || !sites.length) {
            sitesTableBody.innerHTML = '<tr><td colspan="9">Сайтов пока нет</td></tr>';
            sitesCountBadge.textContent = '0 сайтов';
            return;
        }

        sitesCountBadge.textContent = sites.length + ' сайтов';

        var html = '';
        for (var i = 0; i < sites.length; i++) {
            var s = sites[i];
            var status = siteStatus(s);
            var statusClass = status === 'published' ? 'success' : 'warning';

            html += ''
                + '<tr>'
                + '  <td>' + Number(s.id || 0) + '</td>'
                + '  <td>'
                + '    <div class="sb-site-cell">'
                + '      <div class="sb-site-cell__title">' + escapeHtml(s.name || '') + '</div>'
                + '      <div class="sb-site-cell__meta">' + escapeHtml(s.code || '') + '</div>'
                + '    </div>'
                + '  </td>'
                + '  <td>' + escapeHtml(s.slug || '') + '</td>'
                + '  <td><span class="sb-status-badge ' + statusClass + '">' + escapeHtml(status) + '</span></td>'
                + '  <td>' + yesNoBadge(Number(s.topMenuId || 0) > 0) + '</td>'
                + '  <td>' + yesNoBadge(Number(s.diskFolderId || 0) > 0) + '</td>'
                + '  <td>' + escapeHtml(s.createdAt || '—') + '</td>'
                + '  <td>' + escapeHtml(s.updatedAt || '—') + '</td>'
                + '  <td>'
                + '    <a class="sb-btn sb-btn-light sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + Number(s.id || 0) + '">Открыть</a>'
                + '  </td>'
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
                site.code || ''
            ].join(' ').toLowerCase();

            return haystack.indexOf(query) !== -1;
        });

        renderSitesTable(filtered);
    }

    function loadSites() {
        api('site.list', {}, function (res) {
            if (!res || res.ok !== true) {
                sitesTableBody.innerHTML = '<tr><td colspan="9">Не удалось загрузить данные</td></tr>';
                sitesCountBadge.textContent = 'Ошибка загрузки';
                return;
            }

            allSites = Array.isArray(res.sites) ? res.sites : [];
            applySearch();
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

    document.getElementById('createSiteBtn').addEventListener('click', createSite);
    document.getElementById('createSiteQuickBtn').addEventListener('click', function () {
        document.getElementById('siteName').focus();
        window.scrollTo({ top: document.body.scrollHeight, behavior: 'smooth' });
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
```

Если хочешь, следующим сообщением я пришлю еще и **готовый упрощенный `dashboard.css`**, чтобы убрать визуальные стили от уже удаленных блоков статистики.
