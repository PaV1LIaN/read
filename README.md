Отлично. Красоту уже будем наводить после того, как соберём весь рабочий каркас.

Следующий правильный шаг — files.php, чтобы проверить:

file.list

file.upload

file.delete


И заодно сразу поймём, нет ли нюансов по Disk на твоей коробке.

/local/sitebuilder/files.php

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
        <title>SiteBuilder / Files</title>
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
    <title>SiteBuilder / Files</title>
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
            max-width: 1280px;
            margin: 0 auto;
            padding: 24px;
        }
        .topbar {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 16px;
            margin-bottom: 20px;
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
        .back-link {
            display: inline-block;
            margin-bottom: 8px;
            text-decoration: none;
            color: #1d4ed8;
            font-size: 14px;
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
        .toolbar {
            display: flex;
            gap: 10px;
            flex-wrap: wrap;
            align-items: center;
            margin-bottom: 12px;
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
        .meta {
            font-size: 13px;
            color: #6b7280;
            line-height: 1.5;
        }
        .files-list {
            display: flex;
            flex-direction: column;
            gap: 12px;
        }
        .file-card {
            border: 1px solid #e5e7eb;
            border-radius: 12px;
            padding: 14px;
            background: #fafafa;
        }
        .file-head {
            display: flex;
            justify-content: space-between;
            align-items: flex-start;
            gap: 12px;
        }
        .file-title {
            margin: 0 0 6px;
            font-size: 15px;
            font-weight: 700;
            word-break: break-word;
        }
        .file-actions {
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
        .empty {
            padding: 20px;
            text-align: center;
            color: #6b7280;
            border: 1px dashed #d1d5db;
            border-radius: 12px;
            background: #fff;
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
    </style>
</head>
<body>
<div class="page">
    <div class="topbar">
        <div>
            <a class="back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">← К списку сайтов</a>
            <h1 class="title">Файлы сайта</h1>
            <p class="subtitle">siteId = <?= (int)$siteId ?></p>
        </div>
    </div>

    <div class="panel">
        <h2 class="panel-title">Загрузка файла</h2>
        <div class="toolbar">
            <input type="file" id="uploadFileInput">
            <button type="button" class="btn btn-primary" id="uploadBtn">Загрузить</button>
            <button type="button" class="btn btn-light" id="reloadBtn">Обновить список</button>
        </div>
        <div class="meta" id="folderInfo">Папка ещё не загружена</div>
    </div>

    <div class="panel">
        <h2 class="panel-title">Файлы</h2>
        <div id="filesContainer" class="files-list">
            <div class="empty">Загрузка...</div>
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
    var SITE_ID = <?= (int)$siteId ?>;

    var output = document.getElementById('output');
    var filesContainer = document.getElementById('filesContainer');
    var folderInfo = document.getElementById('folderInfo');
    var uploadFileInput = document.getElementById('uploadFileInput');

    var state = {
        files: [],
        folderId: 0
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

    function loadFiles() {
        filesContainer.innerHTML = '<div class="empty">Загрузка...</div>';

        api('file.list', { siteId: SITE_ID }, function (res) {
            if (!res || res.ok !== true) {
                filesContainer.innerHTML = '<div class="empty">Не удалось загрузить файлы</div>';
                folderInfo.textContent = 'Ошибка загрузки папки';
                return;
            }

            state.files = Array.isArray(res.files) ? res.files : [];
            state.folderId = Number(res.folderId || 0);

            folderInfo.textContent = 'Disk folder ID: ' + state.folderId;
            renderFiles();
        });
    }

    function renderFiles() {
        if (!state.files.length) {
            filesContainer.innerHTML = '<div class="empty">Файлов пока нет</div>';
            return;
        }

        var html = '';
        for (var i = 0; i < state.files.length; i++) {
            html += renderFileCard(state.files[i]);
        }
        filesContainer.innerHTML = html;
    }

    function renderFileCard(file) {
        var id = Number(file.id || 0);
        var name = escapeHtml(file.name || '');
        var size = Number(file.size || 0);
        var createTime = escapeHtml(file.createTime || '');
        var updateTime = escapeHtml(file.updateTime || '');
        var downloadUrl = escapeHtml(file.downloadUrl || '#');

        return ''
            + '<div class="file-card">'
            + '  <div class="file-head">'
            + '    <div>'
            + '      <div class="file-title">' + name + '</div>'
            + '      <div class="meta">'
            + '        <div><strong>ID:</strong> ' + id + '</div>'
            + '        <div><strong>Размер:</strong> ' + size + ' байт</div>'
            + '        <div><strong>Создан:</strong> ' + createTime + '</div>'
            + '        <div><strong>Обновлён:</strong> ' + updateTime + '</div>'
            + '      </div>'
            + '    </div>'
            + '    <span class="badge">file</span>'
            + '  </div>'
            + '  <div class="file-actions">'
            + '    <a class="btn btn-light btn-small" href="' + downloadUrl + '" target="_blank">Скачать</a>'
            + '    <button type="button" class="btn btn-danger btn-small js-delete-file" data-id="' + id + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }

    function uploadFile() {
        if (!uploadFileInput.files || !uploadFileInput.files.length) {
            alert('Выберите файл');
            return;
        }

        var file = uploadFileInput.files[0];

        var formData = new FormData();
        formData.append('action', 'file.upload');
        formData.append('siteId', String(SITE_ID));
        formData.append('sessid', getSessid());
        formData.append('file', file);

        var xhr = new XMLHttpRequest();
        xhr.open('POST', API_URL, true);

        xhr.onload = function () {
            var text = xhr.responseText || '';
            var data = null;

            try {
                data = JSON.parse(text);
            } catch (e) {
                print({
                    ok: false,
                    error: 'BAD_JSON',
                    raw: text
                });
                alert('Ответ сервера не является JSON');
                return;
            }

            print(data);

            if (!data || data.ok !== true) {
                alert('Не удалось загрузить файл');
                return;
            }

            uploadFileInput.value = '';
            loadFiles();
        };

        xhr.onerror = function () {
            print({
                ok: false,
                error: 'XHR_UPLOAD_ERROR'
            });
            alert('Ошибка загрузки файла');
        };

        xhr.send(formData);
    }

    function deleteFile(fileId) {
        if (!confirm('Удалить файл #' + fileId + '?')) {
            return;
        }

        api('file.delete', {
            siteId: SITE_ID,
            fileId: fileId
        }, function (res) {
            if (!res || res.ok !== true) {
                alert('Не удалось удалить файл');
                return;
            }

            loadFiles();
        });
    }

    document.getElementById('uploadBtn').addEventListener('click', uploadFile);
    document.getElementById('reloadBtn').addEventListener('click', loadFiles);

    document.addEventListener('click', function (e) {
        var btn = e.target.closest('.js-delete-file');
        if (!btn) {
            return;
        }

        deleteFile(parseInt(btn.getAttribute('data-id'), 10) || 0);
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

    loadFiles();
})();
</script>
</body>
</html>

Что проверить

Открывай:

/local/sitebuilder/files.php?siteId=1

Проверь по порядку:

1. грузится ли список файлов


2. создаётся ли Disk-папка автоматически


3. загружается ли файл


4. появляется ли он в списке


5. работает ли скачивание


6. работает ли удаление



Если будет ошибка

Если на file.list или file.upload появится DISK_ERROR, пришли мне точный текст из debug-блока.
Тогда я подгоню lib/disk.php именно под твою версию Bitrix Disk API.

Что дальше

Если files.php оживёт, следующим шагом логично делать settings.php, а потом уже:

привести layout.php

и только после этого начинать “делать красиво” редактор и интерфейс.


Напиши, что выдал files.php.