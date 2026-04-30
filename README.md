Да, дальше добавляем интерфейс в settings.php.

Ниже два файла:

/local/sitebuilder/settings.php
/local/sitebuilder/assets/admin/settings.css


---

1. /local/sitebuilder/settings.php

Заменяй файл полностью:

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

$libFiles = [
    __DIR__ . '/lib/db.php',
    __DIR__ . '/lib/json.php',
    __DIR__ . '/lib/storage_db.php',
    __DIR__ . '/lib/response.php',
    __DIR__ . '/lib/helpers.php',
    __DIR__ . '/lib/access.php',
];

foreach ($libFiles as $libFile) {
    if (file_exists($libFile)) {
        require_once $libFile;
    }
}

if ($siteId <= 0) {
    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>SiteBuilder / Settings</title>
        <?php $APPLICATION->ShowHead(); ?>
        <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
        <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/settings.css?v=1">
    </head>
    <body class="sb-admin-body">
    <div class="sb-page">
        <h1 class="sb-title">Не передан siteId</h1>
        <p>
            <a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/index.php">
                Вернуться к списку сайтов
            </a>
        </p>
    </div>
    </body>
    </html>
    <?php
    exit;
}

if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}
?>
<!doctype html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <title>SiteBuilder / Settings</title>
    <?php $APPLICATION->ShowHead(); ?>
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/admin.css">
    <link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/settings.css?v=1">
</head>
<body class="sb-admin-body">
<div class="sb-page">
    <div class="sb-topbar">
        <div class="sb-topbar-left">
            <a class="sb-back-link" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/editor.php?siteId=<?= (int)$siteId ?>">
                ← В редактор
            </a>
            <h1 class="sb-title">Настройки сайта</h1>
            <p class="sb-subtitle">siteId = <?= (int)$siteId ?></p>
        </div>

        <div class="sb-settings-top-actions">
            <a class="sb-btn sb-btn-light" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/public.php?siteId=<?= (int)$siteId ?>" target="_blank">
                Открыть публичную
            </a>
            <button class="sb-btn sb-btn-light" type="button" id="reloadBtn">Обновить</button>
        </div>
    </div>

    <div class="sb-settings-layout">
        <div class="sb-settings-main">
            <section class="sb-panel">
                <div class="sb-settings-panel-head">
                    <div>
                        <h2 class="sb-panel-title">Основные настройки</h2>
                        <p class="sb-settings-note">
                            Название, адрес сайта, ширина контейнера и основной цвет.
                        </p>
                    </div>
                </div>

                <div class="sb-settings-grid">
                    <div class="sb-field">
                        <label for="siteNameInput">Название сайта</label>
                        <input class="sb-input" type="text" id="siteNameInput" placeholder="Название">
                    </div>

                    <div class="sb-field">
                        <label for="siteSlugInput">Slug</label>
                        <input class="sb-input" type="text" id="siteSlugInput" placeholder="site-slug">
                    </div>

                    <div class="sb-field">
                        <label for="containerWidthInput">Ширина контейнера</label>
                        <input class="sb-input" type="number" id="containerWidthInput" min="320" max="1920" step="10">
                    </div>

                    <div class="sb-field">
                        <label for="accentInput">Акцентный цвет</label>
                        <input class="sb-input sb-color-input" type="color" id="accentInput" value="#2563eb">
                    </div>
                </div>

                <div class="sb-settings-actions">
                    <button class="sb-btn sb-btn-primary" type="button" id="saveBasicBtn">Сохранить основные настройки</button>
                </div>
            </section>

            <section class="sb-panel">
                <div class="sb-settings-panel-head">
                    <div>
                        <h2 class="sb-panel-title">Логотип</h2>
                        <p class="sb-settings-note">
                            Логотип будет отображаться в шапке публичной части сайта.
                        </p>
                    </div>
                </div>

                <div class="sb-asset-row">
                    <div class="sb-asset-preview sb-asset-preview--logo" id="logoPreview">
                        <span>Нет логотипа</span>
                    </div>

                    <div class="sb-asset-controls">
                        <div class="sb-field">
                            <label for="logoFileInput">Файл логотипа</label>
                            <input class="sb-input" type="file" id="logoFileInput" accept="image/*">
                        </div>

                        <div class="sb-field">
                            <label for="headerLogoModeInput">Отображение в шапке</label>
                            <select class="sb-select" id="headerLogoModeInput">
                                <option value="image">Только логотип</option>
                                <option value="text">Только название сайта</option>
                                <option value="both">Логотип и название</option>
                            </select>
                        </div>

                        <div class="sb-settings-actions">
                            <button class="sb-btn sb-btn-primary" type="button" id="uploadLogoBtn">Загрузить логотип</button>
                            <button class="sb-btn sb-btn-light" type="button" id="removeLogoBtn">Удалить логотип</button>
                        </div>
                    </div>
                </div>
            </section>

            <section class="sb-panel">
                <div class="sb-settings-panel-head">
                    <div>
                        <h2 class="sb-panel-title">Фон сайта</h2>
                        <p class="sb-settings-note">
                            Фон будет применяться к публичной части сайта.
                        </p>
                    </div>
                </div>

                <div class="sb-asset-row">
                    <div class="sb-asset-preview sb-asset-preview--background" id="backgroundPreview">
                        <span>Нет фона</span>
                    </div>

                    <div class="sb-asset-controls">
                        <div class="sb-field">
                            <label for="backgroundFileInput">Изображение фона</label>
                            <input class="sb-input" type="file" id="backgroundFileInput" accept="image/*">
                        </div>

                        <div class="sb-settings-grid">
                            <div class="sb-field">
                                <label for="backgroundColorInput">Цвет фона</label>
                                <input class="sb-input sb-color-input" type="color" id="backgroundColorInput" value="#f8fafc">
                            </div>

                            <div class="sb-field">
                                <label for="backgroundModeInput">Размер</label>
                                <select class="sb-select" id="backgroundModeInput">
                                    <option value="cover">Заполнить экран</option>
                                    <option value="contain">Уместить целиком</option>
                                    <option value="auto">Оригинальный размер</option>
                                    <option value="stretch">Растянуть</option>
                                </select>
                            </div>

                            <div class="sb-field">
                                <label for="backgroundPositionInput">Позиция</label>
                                <select class="sb-select" id="backgroundPositionInput">
                                    <option value="center center">По центру</option>
                                    <option value="top center">Сверху</option>
                                    <option value="bottom center">Снизу</option>
                                    <option value="left center">Слева</option>
                                    <option value="right center">Справа</option>
                                </select>
                            </div>

                            <div class="sb-field">
                                <label for="backgroundRepeatInput">Повтор</label>
                                <select class="sb-select" id="backgroundRepeatInput">
                                    <option value="no-repeat">Не повторять</option>
                                    <option value="repeat">Повторять</option>
                                    <option value="repeat-x">Повторять по X</option>
                                    <option value="repeat-y">Повторять по Y</option>
                                </select>
                            </div>
                        </div>

                        <div class="sb-settings-actions">
                            <button class="sb-btn sb-btn-primary" type="button" id="uploadBackgroundBtn">Загрузить фон</button>
                            <button class="sb-btn sb-btn-light" type="button" id="saveAppearanceBtn">Сохранить настройки фона</button>
                            <button class="sb-btn sb-btn-light" type="button" id="removeBackgroundBtn">Удалить фон</button>
                        </div>
                    </div>
                </div>
            </section>
        </div>

        <aside class="sb-settings-side">
            <section class="sb-panel">
                <h2 class="sb-panel-title">Предпросмотр</h2>

                <div class="sb-appearance-preview" id="appearancePreview">
                    <div class="sb-appearance-preview__header">
                        <div class="sb-appearance-preview__logo" id="previewLogo">S</div>
                        <div class="sb-appearance-preview__title" id="previewTitle">Сайт</div>
                    </div>

                    <div class="sb-appearance-preview__content">
                        <div class="sb-appearance-preview__line"></div>
                        <div class="sb-appearance-preview__line is-short"></div>
                        <button class="sb-appearance-preview__button" type="button">Кнопка</button>
                    </div>
                </div>
            </section>

            <section class="sb-panel">
                <h2 class="sb-panel-title">Статус</h2>
                <div id="settingsMessage" class="sb-empty">Настройки загружаются...</div>
            </section>

            <section class="sb-panel">
                <h2 class="sb-panel-title">Ответ API</h2>
                <div id="output" class="sb-output">Здесь будут ответы API...</div>
            </section>
        </aside>
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
        appearance: null
    };

    var output = document.getElementById('output');
    var message = document.getElementById('settingsMessage');

    function print(data) {
        try {
            output.textContent = typeof data === 'string' ? data : JSON.stringify(data, null, 2);
        } catch (e) {
            output.textContent = String(data);
        }
    }

    function setMessage(text, type) {
        message.classList.remove('is-success', 'is-error');

        if (type === 'success') {
            message.classList.add('is-success');
        }

        if (type === 'error') {
            message.classList.add('is-error');
        }

        message.textContent = text || '';
    }

    function getSessid() {
        if (window.BX && typeof BX.bitrix_sessid === 'function') {
            return BX.bitrix_sessid();
        }

        return '<?= CUtil::JSEscape(bitrix_sessid()) ?>';
    }

    function api(action, data) {
        return fetch(API_URL, {
            method: 'POST',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body: new URLSearchParams(Object.assign({
                action: action,
                sessid: getSessid()
            }, data || {})),
            credentials: 'same-origin'
        }).then(async function (res) {
            var text = await res.text();
            var json = null;

            try {
                json = JSON.parse(text);
            } catch (e) {
                throw {
                    ok: false,
                    error: 'BAD_JSON_RESPONSE',
                    status: res.status,
                    text: text
                };
            }

            print(json);

            if (!json || json.ok !== true) {
                throw json || {ok: false, error: 'UNKNOWN_ERROR'};
            }

            return json;
        });
    }

    function apiUpload(type, file) {
        var fd = new FormData();

        fd.append('action', 'site.appearanceUpload');
        fd.append('sessid', getSessid());
        fd.append('siteId', String(siteId));
        fd.append('type', type);
        fd.append('file', file);

        return fetch(API_URL, {
            method: 'POST',
            body: fd,
            credentials: 'same-origin'
        }).then(async function (res) {
            var text = await res.text();
            var json = null;

            try {
                json = JSON.parse(text);
            } catch (e) {
                throw {
                    ok: false,
                    error: 'BAD_JSON_RESPONSE',
                    status: res.status,
                    text: text
                };
            }

            print(json);

            if (!json || json.ok !== true) {
                throw json || {ok: false, error: 'UNKNOWN_ERROR'};
            }

            return json;
        });
    }

    function getValue(id) {
        var el = document.getElementById(id);
        return el ? String(el.value || '') : '';
    }

    function setValue(id, value) {
        var el = document.getElementById(id);
        if (el) {
            el.value = value == null ? '' : String(value);
        }
    }

    function cssBackgroundSize(mode) {
        if (mode === 'stretch') {
            return '100% 100%';
        }

        if (mode === 'contain') {
            return 'contain';
        }

        if (mode === 'auto') {
            return 'auto';
        }

        return 'cover';
    }

    function renderBasic() {
        var site = state.site || {};
        var settings = site.settings || {};

        setValue('siteNameInput', site.name || '');
        setValue('siteSlugInput', site.slug || '');
        setValue('containerWidthInput', settings.containerWidth || 1100);
        setValue('accentInput', settings.accent || '#2563eb');
    }

    function renderAppearance() {
        var appearance = state.appearance || {};

        setValue('headerLogoModeInput', appearance.headerLogoMode || 'image');
        setValue('backgroundColorInput', appearance.backgroundColor || '#f8fafc');
        setValue('backgroundModeInput', appearance.backgroundMode || 'cover');
        setValue('backgroundPositionInput', appearance.backgroundPosition || 'center center');
        setValue('backgroundRepeatInput', appearance.backgroundRepeat || 'no-repeat');

        renderLogoPreview();
        renderBackgroundPreview();
        renderMainPreview();
    }

    function renderLogoPreview() {
        var appearance = state.appearance || {};
        var node = document.getElementById('logoPreview');

        if (!node) return;

        if (appearance.logoUrl) {
            node.innerHTML = '<img src="' + appearance.logoUrl + '" alt="">';
        } else {
            node.innerHTML = '<span>Нет логотипа</span>';
        }
    }

    function renderBackgroundPreview() {
        var appearance = state.appearance || {};
        var node = document.getElementById('backgroundPreview');

        if (!node) return;

        node.innerHTML = appearance.backgroundUrl ? '' : '<span>Нет фона</span>';
        node.style.backgroundColor = appearance.backgroundColor || '#f8fafc';
        node.style.backgroundImage = appearance.backgroundUrl ? 'url("' + appearance.backgroundUrl + '")' : '';
        node.style.backgroundSize = cssBackgroundSize(appearance.backgroundMode || 'cover');
        node.style.backgroundPosition = appearance.backgroundPosition || 'center center';
        node.style.backgroundRepeat = appearance.backgroundRepeat || 'no-repeat';
    }

    function renderMainPreview() {
        var site = state.site || {};
        var appearance = state.appearance || {};
        var settings = site.settings || {};

        var preview = document.getElementById('appearancePreview');
        var previewLogo = document.getElementById('previewLogo');
        var previewTitle = document.getElementById('previewTitle');

        if (!preview) return;

        var accent = getValue('accentInput') || settings.accent || '#2563eb';

        preview.style.backgroundColor = getValue('backgroundColorInput') || appearance.backgroundColor || '#f8fafc';
        preview.style.backgroundImage = appearance.backgroundUrl ? 'url("' + appearance.backgroundUrl + '")' : '';
        preview.style.backgroundSize = cssBackgroundSize(getValue('backgroundModeInput') || appearance.backgroundMode || 'cover');
        preview.style.backgroundPosition = getValue('backgroundPositionInput') || appearance.backgroundPosition || 'center center';
        preview.style.backgroundRepeat = getValue('backgroundRepeatInput') || appearance.backgroundRepeat || 'no-repeat';
        preview.style.setProperty('--preview-accent', accent);

        previewTitle.textContent = getValue('siteNameInput') || site.name || 'Сайт';

        if (appearance.logoUrl) {
            previewLogo.innerHTML = '<img src="' + appearance.logoUrl + '" alt="">';
        } else {
            var title = getValue('siteNameInput') || site.name || 'S';
            previewLogo.textContent = title.substring(0, 1).toUpperCase();
        }
    }

    async function loadAll() {
        setMessage('Загружаю настройки...', '');

        var siteRes = await api('site.get', {
            siteId: siteId
        });

        state.site = siteRes.site || null;

        var appearanceRes = await api('site.appearanceGet', {
            siteId: siteId
        });

        state.appearance = appearanceRes.appearance || {};

        renderBasic();
        renderAppearance();

        setMessage('Настройки загружены', 'success');
    }

    async function saveBasic() {
        var appearance = state.appearance || {};

        setMessage('Сохраняю основные настройки...', '');

        var res = await api('site.update', {
            siteId: siteId,
            name: getValue('siteNameInput').trim(),
            slug: getValue('siteSlugInput').trim(),
            containerWidth: getValue('containerWidthInput') || '1100',
            accent: getValue('accentInput') || '#2563eb',

            /*
             * Важно: site.update в старом обработчике принимает logoFileId.
             * Если его не передать, можно случайно сбросить логотип.
             */
            logoFileId: appearance.logoFileId || 0
        });

        state.site = res.site || state.site;

        renderBasic();
        renderMainPreview();

        setMessage('Основные настройки сохранены', 'success');
    }

    async function saveAppearance() {
        setMessage('Сохраняю оформление...', '');

        var res = await api('site.appearanceUpdate', {
            siteId: siteId,
            backgroundColor: getValue('backgroundColorInput') || '#f8fafc',
            backgroundMode: getValue('backgroundModeInput') || 'cover',
            backgroundPosition: getValue('backgroundPositionInput') || 'center center',
            backgroundRepeat: getValue('backgroundRepeatInput') || 'no-repeat',
            headerLogoMode: getValue('headerLogoModeInput') || 'image'
        });

        state.appearance = res.appearance || state.appearance;

        renderAppearance();

        setMessage('Оформление сохранено', 'success');
    }

    async function uploadLogo() {
        var input = document.getElementById('logoFileInput');

        if (!input || !input.files || !input.files[0]) {
            alert('Выбери файл логотипа');
            return;
        }

        setMessage('Загружаю логотип...', '');

        var res = await apiUpload('logo', input.files[0]);

        state.appearance = res.appearance || state.appearance;
        input.value = '';

        renderAppearance();

        setMessage('Логотип загружен', 'success');
    }

    async function uploadBackground() {
        var input = document.getElementById('backgroundFileInput');

        if (!input || !input.files || !input.files[0]) {
            alert('Выбери изображение фона');
            return;
        }

        setMessage('Загружаю фон...', '');

        var res = await apiUpload('background', input.files[0]);

        state.appearance = res.appearance || state.appearance;
        input.value = '';

        renderAppearance();

        setMessage('Фон загружен', 'success');
    }

    async function removeLogo() {
        if (!confirm('Удалить логотип?')) {
            return;
        }

        setMessage('Удаляю логотип...', '');

        var res = await api('site.appearanceRemove', {
            siteId: siteId,
            type: 'logo'
        });

        state.appearance = res.appearance || state.appearance;

        renderAppearance();

        setMessage('Логотип удалён', 'success');
    }

    async function removeBackground() {
        if (!confirm('Удалить фон?')) {
            return;
        }

        setMessage('Удаляю фон...', '');

        var res = await api('site.appearanceRemove', {
            siteId: siteId,
            type: 'background'
        });

        state.appearance = res.appearance || state.appearance;

        renderAppearance();

        setMessage('Фон удалён', 'success');
    }

    document.getElementById('reloadBtn').addEventListener('click', function () {
        loadAll().catch(function (e) {
            print(e);
            setMessage('Ошибка загрузки настроек: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
        });
    });

    document.getElementById('saveBasicBtn').addEventListener('click', function () {
        saveBasic().catch(function (e) {
            print(e);
            setMessage('Ошибка сохранения основных настроек: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
        });
    });

    document.getElementById('saveAppearanceBtn').addEventListener('click', function () {
        saveAppearance().catch(function (e) {
            print(e);
            setMessage('Ошибка сохранения оформления: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
        });
    });

    document.getElementById('uploadLogoBtn').addEventListener('click', function () {
        uploadLogo().catch(function (e) {
            print(e);
            setMessage('Ошибка загрузки логотипа: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
        });
    });

    document.getElementById('uploadBackgroundBtn').addEventListener('click', function () {
        uploadBackground().catch(function (e) {
            print(e);
            setMessage('Ошибка загрузки фона: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
        });
    });

    document.getElementById('removeLogoBtn').addEventListener('click', function () {
        removeLogo().catch(function (e) {
            print(e);
            setMessage('Ошибка удаления логотипа: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
        });
    });

    document.getElementById('removeBackgroundBtn').addEventListener('click', function () {
        removeBackground().catch(function (e) {
            print(e);
            setMessage('Ошибка удаления фона: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
        });
    });

    [
        'siteNameInput',
        'accentInput',
        'backgroundColorInput',
        'backgroundModeInput',
        'backgroundPositionInput',
        'backgroundRepeatInput'
    ].forEach(function (id) {
        var node = document.getElementById(id);
        if (node) {
            node.addEventListener('input', renderMainPreview);
            node.addEventListener('change', renderMainPreview);
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

    loadAll().catch(function (e) {
        print(e);
        setMessage('Ошибка загрузки настроек: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    });
})();
</script>
</body>
</html>


---

2. /local/sitebuilder/assets/admin/settings.css

Создай файл:

/local/sitebuilder/assets/admin/settings.css

.sb-settings-top-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    justify-content: flex-end;
}

.sb-settings-layout {
    display: grid;
    grid-template-columns: minmax(0, 1fr) 360px;
    gap: 20px;
    align-items: start;
}

.sb-settings-main {
    display: flex;
    flex-direction: column;
    gap: 16px;
    min-width: 0;
}

.sb-settings-side {
    display: flex;
    flex-direction: column;
    gap: 16px;
    min-width: 0;
    position: sticky;
    top: 16px;
}

.sb-settings-panel-head {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    margin-bottom: 14px;
}

.sb-settings-note {
    margin: 6px 0 0;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.5;
}

.sb-settings-grid {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 12px;
}

.sb-settings-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 14px;
}

.sb-color-input {
    min-height: 42px;
    padding: 4px;
}

.sb-asset-row {
    display: grid;
    grid-template-columns: 260px minmax(0, 1fr);
    gap: 16px;
    align-items: start;
}

.sb-asset-preview {
    min-height: 160px;
    border: 1px dashed #d1d5db;
    border-radius: 16px;
    background: #f8fafc;
    color: #6b7280;
    display: flex;
    align-items: center;
    justify-content: center;
    overflow: hidden;
}

.sb-asset-preview span {
    font-size: 13px;
}

.sb-asset-preview--logo {
    min-height: 120px;
    padding: 18px;
}

.sb-asset-preview--logo img {
    max-width: 100%;
    max-height: 90px;
    object-fit: contain;
    display: block;
}

.sb-asset-preview--background {
    min-height: 180px;
    background-position: center center;
    background-repeat: no-repeat;
    background-size: cover;
}

.sb-asset-controls {
    min-width: 0;
}

.sb-appearance-preview {
    min-height: 260px;
    border: 1px solid #e5e7eb;
    border-radius: 18px;
    overflow: hidden;
    background-color: #f8fafc;
    background-position: center center;
    background-repeat: no-repeat;
    background-size: cover;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.08);
}

.sb-appearance-preview__header {
    min-height: 64px;
    padding: 14px 16px;
    background: rgba(255, 255, 255, 0.88);
    display: flex;
    align-items: center;
    gap: 10px;
    border-bottom: 1px solid rgba(229, 231, 235, .8);
}

.sb-appearance-preview__logo {
    width: 38px;
    height: 38px;
    border-radius: 12px;
    background: var(--preview-accent, #2563eb);
    color: #fff;
    display: flex;
    align-items: center;
    justify-content: center;
    font-weight: 800;
    overflow: hidden;
}

.sb-appearance-preview__logo img {
    width: 100%;
    height: 100%;
    object-fit: contain;
    display: block;
    background: #fff;
}

.sb-appearance-preview__title {
    font-size: 15px;
    font-weight: 800;
    color: #111827;
    min-width: 0;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-appearance-preview__content {
    margin: 26px 18px;
    padding: 18px;
    border-radius: 16px;
    background: rgba(255, 255, 255, .88);
    border: 1px solid rgba(229, 231, 235, .8);
}

.sb-appearance-preview__line {
    height: 12px;
    border-radius: 999px;
    background: #d1d5db;
    margin-bottom: 10px;
}

.sb-appearance-preview__line.is-short {
    width: 68%;
}

.sb-appearance-preview__button {
    margin-top: 8px;
    border: 0;
    border-radius: 10px;
    min-height: 34px;
    padding: 0 14px;
    background: var(--preview-accent, #2563eb);
    color: #fff;
    font-weight: 700;
}

#settingsMessage {
    line-height: 1.45;
    white-space: pre-line;
}

#settingsMessage.is-success {
    border-color: #bbf7d0;
    background: #f0fdf4;
    color: #166534;
}

#settingsMessage.is-error {
    border-color: #fecaca;
    background: #fef2f2;
    color: #991b1b;
}

@media (max-width: 1180px) {
    .sb-settings-layout {
        grid-template-columns: 1fr;
    }

    .sb-settings-side {
        position: static;
    }

    .sb-asset-row {
        grid-template-columns: 220px minmax(0, 1fr);
    }
}

@media (max-width: 760px) {
    .sb-settings-top-actions {
        justify-content: flex-start;
    }

    .sb-settings-grid {
        grid-template-columns: 1fr;
    }

    .sb-asset-row {
        grid-template-columns: 1fr;
    }
}


---

После этого открой:

/local/sitebuilder/settings.php?siteId=ID_САЙТА

Проверь:

1. Загружается страница настроек.
2. Работает загрузка логотипа.
3. Работает удаление логотипа.
4. Работает загрузка фона.
5. Работает удаление фона.
6. Сохраняются цвет, режим фона, позиция и повтор.

Если это заработает, следующим шагом подключим логотип и фон в публичной части сайта.