Отлично. Делаем первый рабочий index.php для новой структуры.

Он будет:

показывать список сайтов

создавать сайт

удалять сайт

давать ссылки на:

editor

layout

menu

files

settings



Пока без лишней красоты, но уже рабочий и нормальный по структуре.


---

/local/sitebuilder/index.php

<?php
require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

global $APPLICATION, $USER;

if (!$USER->IsAuthorized()) {
    require $_SERVER['DOCUMENT_ROOT'] . '/auth.php';
    exit;
}

CJSCore::Init(['ajax']);

header('Content-Type: text/html; charset=UTF-8');

$basePath = '/local/sitebuilder';
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder</title>
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
            max-width: 1200px;
            margin: 0 auto;
            padding: 24px;
        }
        .topbar {
            display: flex;
            align-items: center;
            justify-content: space-between;
            gap: 16px;
            margin-bottom: 24px;
        }
        .title {
            margin: 0;
            font-size: 28px;
            font-weight: 700;
        }
        .subtitle {
            margin: 6px 0 0;
            color: #6b7280;
            font-size: 14px;
        }
        .userbox {
            font-size: 14px;
            color: #374151;
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 10px;
            padding: 10px 14px;
        }
        .panel {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 18px;
            margin-bottom: 20px;
            box-shadow: 0 2px 8px rgba(15, 23, 42, 0.04);
        }
        .panel-title {
            margin: 0 0 14px;
            font-size: 18px;
            font-weight: 700;
        }
        .form-row {
            display: flex;
            gap: 12px;
            flex-wrap: wrap;
            align-items: end;
        }
        .field {
            display: flex;
            flex-direction: column;
            gap: 6px;
            min-width: 240px;
            flex: 1 1 240px;
        }
        .field label {
            font-size: 13px;
            color: #4b5563;
        }
        .field input {
            width: 100%;
            height: 40px;
            padding: 0 12px;
            border: 1px solid #d1d5db;
            border-radius: 10px;
            outline: none;
            background: #fff;
        }
        .field input:focus {
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
        .btn-primary {
            background: #2563eb;
            color: #fff;
        }
        .btn-primary:hover {
            background: #1d4ed8;
        }
        .btn-danger {
            background: #dc2626;
            color: #fff;
        }
        .btn-danger:hover {
            background: #b91c1c;
        }
        .btn-light {
            background: #eef2ff;
            color: #1e3a8a;
        }
        .btn-light:hover {
            background: #e0e7ff;
        }
        .btn-small {
            height: 34px;
            padding: 0 12px;
            font-size: 13px;
        }
        .muted {
            color: #6b7280;
        }
        .sites-grid {
            display: grid;
            grid-template-columns: repeat(auto-fill, minmax(340px, 1fr));
            gap: 16px;
        }
        .site-card {
            background: #fff;
            border: 1px solid #e5e7eb;
            border-radius: 14px;
            padding: 16px;
            display: flex;
            flex-direction: column;
            gap: 12px;
        }
        .site-head {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 12px;
        }
        .site-name {
            margin: 0;
            font-size: 18px;
            font-weight: 700;
            word-break: break-word;
        }
        .site-meta {
            font-size: 13px;
            color: #6b7280;
            line-height: 1.5;
        }
        .site-actions {
            display: flex;
            flex-wrap: wrap;
            gap: 8px;
        }
        .site-actions a,
        .site-actions button {
            text-decoration: none;
        }
        .status {
            display: inline-flex;
            align-items: center;
            gap: 6px;
            background: #f3f4f6;
            color: #374151;
            border-radius: 999px;
            padding: 4px 10px;
            font-size: 12px;
            white-space: nowrap;
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
        .empty {
            padding: 24px;
            text-align: center;
            color: #6b7280;
            border: 1px dashed #d1d5db;
            border-radius: 12px;
            background: #fff;
        }
        @media (max-width: 768px) {
            .topbar {
                flex-direction: column;
                align-items: stretch;
            }
            .sites-grid {
                grid-template-columns: 1fr;
            }
        }
    </style>
</head>
<body>
<div class="page">
    <div class="topbar">
        <div>
            <h1 class="title">SiteBuilder</h1>
            <p class="subtitle">Управление сайтами конструктора</p>
        </div>
        <div class="userbox">
            Пользователь:
            <strong><?= htmlspecialchars((string)$USER->GetLogin(), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?></strong>
            (ID <?= (int)$USER->GetID() ?>)
        </div>
    </div>

    <div class="panel">
        <h2 class="panel-title">Создать сайт</h2>
        <div class="form-row">
            <div class="field">
                <label for="siteName">Название сайта</label>
                <input type="text" id="siteName" placeholder="Например: Корпоративный портал">
            </div>
            <div class="field">
                <label for="siteSlug">Slug</label>
                <input type="text" id="siteSlug" placeholder="Например: corp-portal">
            </div>
            <button type="button" class="btn btn-primary" id="createSiteBtn">Создать</button>
        </div>
    </div>

    <div class="panel">
        <div style="display:flex; justify-content:space-between; align-items:center; gap:12px; margin-bottom:14px;">
            <h2 class="panel-title" style="margin:0;">Сайты</h2>
            <button type="button" class="btn btn-light btn-small" id="reloadBtn">Обновить список</button>
        </div>
        <div id="sitesContainer">
            <div class="empty">Загрузка списка сайтов...</div>
        </div>
    </div>

    <div class="panel">
        <h2 class="panel-title">Отладка</h2>
        <div id="output" class="output">Здесь будут ответы API...</div>
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
            + '<div class="site-card">'
            + '  <div class="site-head">'
            + '    <div>'
            + '      <h3 class="site-name">' + name + '</h3>'
            + '      <div class="site-meta">'
            + '        <div><strong>ID:</strong> ' + id + '</div>'
            + '        <div><strong>Slug:</strong> ' + slug + '</div>'
            + '        <div><strong>Home page ID:</strong> ' + homePageId + '</div>'
            + '        <div><strong>Disk folder ID:</strong> ' + diskFolderId + '</div>'
            + '        <div><strong>Создан:</strong> ' + createdAt + '</div>'
            + '      </div>'
            + '    </div>'
            + '    <span class="status">site #' + id + '</span>'
            + '  </div>'
            + ''
            + '  <div class="site-actions">'
            + '    <a class="btn btn-light btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + id + '">Редактор</a>'
            + '    <a class="btn btn-light btn-small" href="' + BASE_PATH + '/layout.php?siteId=' + id + '">Layout</a>'
            + '    <a class="btn btn-light btn-small" href="' + BASE_PATH + '/menu.php?siteId=' + id + '">Меню</a>'
            + '    <a class="btn btn-light btn-small" href="' + BASE_PATH + '/files.php?siteId=' + id + '">Файлы</a>'
            + '    <a class="btn btn-light btn-small" href="' + BASE_PATH + '/settings.php?siteId=' + id + '">Настройки</a>'
            + '    <a class="btn btn-light btn-small" href="' + BASE_PATH + '/public.php?siteId=' + id + '" target="_blank">Публичная</a>'
            + '    <button type="button" class="btn btn-danger btn-small js-delete-site" data-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }

    function renderSites(sites) {
        if (!Array.isArray(sites) || !sites.length) {
            sitesContainer.innerHTML = '<div class="empty">Сайтов пока нет</div>';
            return;
        }

        var html = '<div class="sites-grid">';
        for (var i = 0; i < sites.length; i++) {
            html += siteCard(sites[i]);
        }
        html += '</div>';

        sitesContainer.innerHTML = html;
    }

    function loadSites() {
        sitesContainer.innerHTML = '<div class="empty">Загрузка...</div>';

        api('site.list', {}, function (res) {
            if (!res || res.ok !== true) {
                sitesContainer.innerHTML = '<div class="empty">Не удалось загрузить список сайтов</div>';
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


---

Что должно быть по ссылке

Если папка у тебя:

/local/sitebuilder/

то открыть нужно:

https://ТВОЙ_ДОМЕН/local/sitebuilder/index.php


---

Что он уже умеет

Сразу после открытия:

вызовет site.list

покажет карточки сайтов


При создании:

вызовет site.create

обновит список


При удалении:

вызовет site.delete

обновит список



---

Что важно

1. Пути

Если папка называется не sitebuilder, а иначе, поменяй здесь:

$basePath = '/local/sitebuilder';

Например:

$basePath = '/local/sitebuilder_new';


---

2. Ссылки на остальные страницы

Сейчас ссылки уже есть:

editor.php

layout.php

menu.php

files.php

settings.php

public.php


Но если этих файлов ещё нет или они пустые — это нормально. index.php уже можно проверять отдельно.


---

Что делать следующим шагом

Теперь логично сделать следующие страницы-оболочки:

1. editor.php


2. menu.php


3. files.php


4. settings.php


5. layout.php



Я бы следующим сообщением сделал menu.php, потому что он проще редактора и быстро даст рабочую административную страницу.

Напиши: давай menu.php