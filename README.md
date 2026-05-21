Заменяй полностью файл:

/local/sitebuilder/components/disk/script.js

на этот:

/* =========================================================
   SITEBUILDER DISK
   Modern File Manager UI
   ========================================================= */

(function () {
    'use strict';

    var currentScript = document.currentScript || (function () {
        var scripts = document.getElementsByTagName('script');
        return scripts[scripts.length - 1];
    })();

    var SCRIPT_SRC = currentScript && currentScript.src ? currentScript.src : '';
    var COMPONENT_URL = SCRIPT_SRC.split('/script.js')[0] || '/local/sitebuilder/components/disk';
    var API_URL = COMPONENT_URL + '/api.php';

    function h(value) {
        return String(value == null ? '' : value)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#039;');
    }

    function lower(value) {
        return String(value || '').toLowerCase();
    }

    function getSessid(root) {
        if (root && root.getAttribute('data-sessid')) {
            return root.getAttribute('data-sessid');
        }

        if (window.BX && typeof BX.bitrix_sessid === 'function') {
            return BX.bitrix_sessid();
        }

        return '';
    }

    function getFileExtension(name) {
        name = String(name || '').trim();

        var cleanName = name.split('?')[0].split('#')[0];
        var parts = cleanName.split('.');

        if (parts.length < 2) {
            return '';
        }

        return lower(parts.pop());
    }

    function normalizeItem(raw) {
        raw = raw || {};

        var name = String(
            raw.name ||
            raw.NAME ||
            raw.title ||
            raw.TITLE ||
            'Без названия'
        );

        var type = lower(
            raw.type ||
            raw.TYPE ||
            raw.objectType ||
            raw.OBJECT_TYPE ||
            raw.kind ||
            ''
        );

        var isFolder = !!(
            raw.isFolder ||
            raw.IS_FOLDER ||
            type === 'folder' ||
            type === 'dir' ||
            type === 'directory'
        );

        var id = Number(
            raw.id ||
            raw.ID ||
            raw.objectId ||
            raw.OBJECT_ID ||
            raw.fileId ||
            raw.FILE_ID ||
            raw.folderId ||
            raw.FOLDER_ID ||
            0
        );

        var ext = isFolder ? '' : getFileExtension(name);

        return {
            id: id,
            raw: raw,
            name: name,
            type: isFolder ? 'folder' : 'file',
            isFolder: isFolder,
            ext: ext,
            size: raw.size || raw.SIZE || '',
            sizeText: raw.sizeText || raw.SIZE_TEXT || raw.sizeFormatted || raw.SIZE_FORMATTED || '',
            updatedAt: raw.updatedAt || raw.UPDATED_AT || raw.modifiedAt || raw.MODIFIED_AT || raw.updateTime || raw.UPDATE_TIME || '',
            downloadUrl: raw.downloadUrl || raw.DOWNLOAD_URL || raw.url || raw.URL || '',
            viewUrl: raw.viewUrl || raw.VIEW_URL || '',
            canDelete: raw.canDelete !== false && raw.CAN_DELETE !== false,
            canRename: raw.canRename !== false && raw.CAN_RENAME !== false,
            canDownload: raw.canDownload !== false && raw.CAN_DOWNLOAD !== false
        };
    }

    function iconText(item) {
        if (item.isFolder) {
            return '📁';
        }

        if (item.ext === 'pdf') return 'PDF';
        if (['doc', 'docx', 'rtf'].indexOf(item.ext) !== -1) return 'DOC';
        if (['xls', 'xlsx', 'csv'].indexOf(item.ext) !== -1) return 'XLS';
        if (['ppt', 'pptx'].indexOf(item.ext) !== -1) return 'PPT';
        if (['jpg', 'jpeg', 'png', 'gif', 'webp', 'svg'].indexOf(item.ext) !== -1) return 'IMG';
        if (['zip', 'rar', '7z'].indexOf(item.ext) !== -1) return 'ZIP';
        if (['txt', 'log'].indexOf(item.ext) !== -1) return 'TXT';

        return 'FILE';
    }

    function iconClass(item) {
        if (item.isFolder) return 'sb-disk-icon sb-disk-icon-folder';
        if (item.ext === 'pdf') return 'sb-disk-icon sb-disk-icon-pdf';
        if (['doc', 'docx', 'rtf'].indexOf(item.ext) !== -1) return 'sb-disk-icon sb-disk-icon-doc';
        if (['xls', 'xlsx', 'csv'].indexOf(item.ext) !== -1) return 'sb-disk-icon sb-disk-icon-xls';
        if (['ppt', 'pptx'].indexOf(item.ext) !== -1) return 'sb-disk-icon sb-disk-icon-ppt';
        if (['jpg', 'jpeg', 'png', 'gif', 'webp', 'svg'].indexOf(item.ext) !== -1) return 'sb-disk-icon sb-disk-icon-img';
        if (['zip', 'rar', '7z'].indexOf(item.ext) !== -1) return 'sb-disk-icon sb-disk-icon-zip';

        return 'sb-disk-icon sb-disk-icon-file';
    }

    function formatSize(item) {
        if (item.isFolder) {
            return 'Папка';
        }

        if (item.sizeText) {
            return item.sizeText;
        }

        var bytes = Number(item.size || 0);

        if (!bytes) {
            return '—';
        }

        var units = ['Б', 'КБ', 'МБ', 'ГБ'];
        var index = 0;

        while (bytes >= 1024 && index < units.length - 1) {
            bytes = bytes / 1024;
            index++;
        }

        return (index === 0 ? bytes : bytes.toFixed(1)) + ' ' + units[index];
    }

    function api(root, action, data) {
        data = data || {};

        var params = new URLSearchParams();

        params.append('action', action);
        params.append('sessid', getSessid(root));
        params.append('siteId', root.getAttribute('data-site-id') || '0');
        params.append('pageId', root.getAttribute('data-page-id') || '0');
        params.append('blockId', root.getAttribute('data-block-id') || '0');

        Object.keys(data).forEach(function (key) {
            params.append(key, data[key]);
        });

        return fetch(API_URL, {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body: params
        }).then(function (res) {
            return res.text().then(function (text) {
                var json;

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

                if (!json || json.ok !== true) {
                    throw json || { ok: false, error: 'UNKNOWN_ERROR' };
                }

                return json;
            });
        });
    }

    function apiUpload(root, folderId, file) {
        var fd = new FormData();

        fd.append('action', 'upload');
        fd.append('sessid', getSessid(root));
        fd.append('siteId', root.getAttribute('data-site-id') || '0');
        fd.append('pageId', root.getAttribute('data-page-id') || '0');
        fd.append('blockId', root.getAttribute('data-block-id') || '0');
        fd.append('folderId', String(folderId || 0));
        fd.append('file', file);

        return fetch(API_URL, {
            method: 'POST',
            credentials: 'same-origin',
            body: fd
        }).then(function (res) {
            return res.text().then(function (text) {
                var json;

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

                if (!json || json.ok !== true) {
                    throw json || { ok: false, error: 'UNKNOWN_ERROR' };
                }

                return json;
            });
        });
    }

    function getPayload(res) {
        return res.data || res.result || res;
    }

    function buildApp(root) {
        root.innerHTML = ''
            + '<div class="sb-disk-app">'
            + '  <div class="sb-disk-head">'
            + '    <div>'
            + '      <h3 class="sb-disk-title">Файлы</h3>'
            + '      <div class="sb-disk-subtitle" data-role="stats">Загрузка...</div>'
            + '    </div>'
            + '    <div class="sb-disk-head-actions">'
            + '      <button type="button" class="sb-disk-btn" data-action="refresh">Обновить</button>'
            + '    </div>'
            + '  </div>'
            + ''
            + '  <div class="sb-disk-breadcrumbs" data-role="breadcrumbs"></div>'
            + ''
            + '  <div class="sb-disk-modern-panel">'
            + '    <div class="sb-disk-modern-row">'
            + '      <div class="sb-disk-modern-filters">'
            + '        <input class="sb-disk-modern-search" type="search" data-role="search" placeholder="Поиск файлов и папок">'
            + '        <select class="sb-disk-modern-select" data-role="sort">'
            + '          <option value="new">Сначала новые</option>'
            + '          <option value="old">Сначала старые</option>'
            + '          <option value="name">По названию</option>'
            + '          <option value="type">По типу</option>'
            + '        </select>'
            + '      </div>'
            + '      <div class="sb-disk-modern-actions">'
            + '        <button type="button" class="sb-disk-modern-btn sb-disk-modern-btn-primary" data-action="upload">Загрузить</button>'
            + '        <button type="button" class="sb-disk-modern-btn" data-action="create-folder">Новая папка</button>'
            + '        <button type="button" class="sb-disk-modern-btn sb-disk-modern-btn-view is-active" data-action="view-table">Таблица</button>'
            + '        <button type="button" class="sb-disk-modern-btn sb-disk-modern-btn-view" data-action="view-grid">Плитка</button>'
            + '      </div>'
            + '    </div>'
            + '  </div>'
            + ''
            + '  <input type="file" data-role="file-input" style="display:none" multiple>'
            + '  <div class="sb-disk-message" data-role="message" style="display:none"></div>'
            + '  <div class="sb-disk-content" data-role="content"></div>'
            + '</div>';
    }

    function createState(root) {
        return {
            root: root,
            currentFolderId: 0,
            rootFolderId: 0,
            items: [],
            breadcrumbs: [],
            permissions: {},
            view: 'table',
            query: '',
            sort: 'new',
            loading: false
        };
    }

    function setMessage(state, text, type) {
        var node = state.root.querySelector('[data-role="message"]');

        if (!node) return;

        if (!text) {
            node.style.display = 'none';
            node.textContent = '';
            node.className = 'sb-disk-message';
            return;
        }

        node.style.display = 'block';
        node.textContent = text;
        node.className = 'sb-disk-message ' + (type ? 'is-' + type : '');
    }

    function setLoading(state, loading) {
        state.loading = !!loading;

        var content = state.root.querySelector('[data-role="content"]');

        if (loading && content) {
            content.innerHTML = ''
                + '<div class="sb-disk-empty-enhanced">'
                + '  <div class="sb-disk-empty-icon">📁</div>'
                + '  <strong>Загружаю файлы...</strong>'
                + '  <span>Подождите несколько секунд.</span>'
                + '</div>';
        }
    }

    function updateStats(state) {
        var stats = state.root.querySelector('[data-role="stats"]');

        if (!stats) return;

        var folders = state.items.filter(function (item) {
            return item.isFolder;
        }).length;

        var files = state.items.length - folders;

        stats.textContent = files + ' файлов · ' + folders + ' папок';
    }

    function renderBreadcrumbs(state) {
        var node = state.root.querySelector('[data-role="breadcrumbs"]');

        if (!node) return;

        var crumbs = state.breadcrumbs || [];

        if (!crumbs.length) {
            node.innerHTML = '<button type="button" data-folder-id="' + state.rootFolderId + '">Файлы</button>';
            return;
        }

        node.innerHTML = crumbs.map(function (crumb, index) {
            var id = Number(crumb.id || crumb.ID || crumb.folderId || crumb.FOLDER_ID || state.rootFolderId || 0);
            var name = String(crumb.name || crumb.NAME || crumb.title || crumb.TITLE || (index === 0 ? 'Файлы' : 'Папка'));

            return '<button type="button" data-folder-id="' + id + '">' + h(name) + '</button>';
        }).join('<span class="sb-disk-breadcrumb-separator">/</span>');
    }

    function getVisibleItems(state) {
        var query = lower(state.query);
        var items = state.items.slice();

        if (query) {
            items = items.filter(function (item) {
                return lower(item.name).indexOf(query) !== -1;
            });
        }

        items.sort(function (a, b) {
            if (a.isFolder !== b.isFolder) {
                return a.isFolder ? -1 : 1;
            }

            if (state.sort === 'name') {
                return a.name.localeCompare(b.name, 'ru');
            }

            if (state.sort === 'type') {
                return (a.ext || a.type).localeCompare((b.ext || b.type), 'ru');
            }

            var da = new Date(a.updatedAt || 0).getTime();
            var db = new Date(b.updatedAt || 0).getTime();

            if (state.sort === 'old') {
                return da - db;
            }

            return db - da;
        });

        return items;
    }

    function renderEmpty(state) {
        return ''
            + '<div class="sb-disk-empty-enhanced">'
            + '  <div class="sb-disk-empty-icon">📁</div>'
            + '  <strong>Пока здесь пусто</strong>'
            + '  <span>Загрузите первый файл или создайте новую папку.</span>'
            + '  <div class="sb-disk-empty-actions">'
            + '    <button type="button" class="sb-disk-empty-upload" data-action="upload">Загрузить файл</button>'
            + '  </div>'
            + '</div>';
    }

    function renderTable(state, items) {
        if (!items.length) {
            return renderEmpty(state);
        }

        return ''
            + '<div class="sb-disk-table-wrap">'
            + '  <table class="sb-disk-table">'
            + '    <thead>'
            + '      <tr>'
            + '        <th>Название</th>'
            + '        <th>Тип</th>'
            + '        <th>Размер</th>'
            + '        <th>Изменён</th>'
            + '        <th></th>'
            + '      </tr>'
            + '    </thead>'
            + '    <tbody>'
            + items.map(function (item) {
                var typeText = item.isFolder ? 'Папка' : (item.ext ? item.ext.toUpperCase() : 'Файл');

                return ''
                    + '<tr data-item-id="' + item.id + '" data-item-type="' + h(item.type) + '" data-ext="' + h(item.ext) + '">'
                    + '  <td>'
                    + '    <button type="button" class="sb-disk-name-button" data-action="' + (item.isFolder ? 'open-folder' : 'open-file') + '" data-item-id="' + item.id + '">'
                    + '      <span class="' + iconClass(item) + '">' + h(iconText(item)) + '</span>'
                    + '      <span class="sb-disk-name-label">' + h(item.name) + '</span>'
                    + '    </button>'
                    + '  </td>'
                    + '  <td>' + h(typeText) + '</td>'
                    + '  <td>' + h(formatSize(item)) + '</td>'
                    + '  <td>' + h(item.updatedAt || '—') + '</td>'
                    + '  <td class="sb-disk-row-actions">'
                    + renderItemActions(item)
                    + '  </td>'
                    + '</tr>';
            }).join('')
            + '    </tbody>'
            + '  </table>'
            + '</div>';
    }

    function renderGrid(state, items) {
        if (!items.length) {
            return renderEmpty(state);
        }

        return ''
            + '<div class="sb-disk-grid">'
            + items.map(function (item) {
                return ''
                    + '<div class="sb-disk-card" data-item-id="' + item.id + '" data-item-type="' + h(item.type) + '" data-ext="' + h(item.ext) + '">'
                    + '  <button type="button" class="sb-disk-card-open" data-action="' + (item.isFolder ? 'open-folder' : 'open-file') + '" data-item-id="' + item.id + '">'
                    + '    <div class="sb-disk-card-preview">'
                    + '      <span class="' + iconClass(item) + '">' + h(iconText(item)) + '</span>'
                    + '    </div>'
                    + '    <div class="sb-disk-card-title">' + h(item.name) + '</div>'
                    + '    <div class="sb-disk-card-meta">' + h(item.isFolder ? 'Папка' : formatSize(item)) + '</div>'
                    + '  </button>'
                    + '  <div class="sb-disk-card-actions">' + renderItemActions(item) + '</div>'
                    + '</div>';
            }).join('')
            + '</div>';
    }

    function renderItemActions(item) {
        var html = '';

        if (!item.isFolder && item.canDownload) {
            html += '<button type="button" class="sb-disk-mini-btn" data-action="download" data-item-id="' + item.id + '">Скачать</button>';
        }

        if (item.canRename) {
            html += '<button type="button" class="sb-disk-mini-btn" data-action="rename" data-item-id="' + item.id + '">Переименовать</button>';
        }

        if (item.canDelete) {
            html += '<button type="button" class="sb-disk-mini-btn is-danger" data-action="delete" data-item-id="' + item.id + '">Удалить</button>';
        }

        return html;
    }

    function render(state) {
        updateStats(state);
        renderBreadcrumbs(state);

        var content = state.root.querySelector('[data-role="content"]');
        var tableBtn = state.root.querySelector('[data-action="view-table"]');
        var gridBtn = state.root.querySelector('[data-action="view-grid"]');

        if (tableBtn) tableBtn.classList.toggle('is-active', state.view === 'table');
        if (gridBtn) gridBtn.classList.toggle('is-active', state.view === 'grid');

        if (!content) return;

        var items = getVisibleItems(state);

        content.innerHTML = state.view === 'grid'
            ? renderGrid(state, items)
            : renderTable(state, items);
    }

    function loadBootstrap(state) {
        setLoading(state, true);
        setMessage(state, '', '');

        return api(state.root, 'bootstrap', {
            folderId: state.currentFolderId || 0
        }).then(function (res) {
            var payload = getPayload(res);

            state.rootFolderId = Number(payload.rootFolderId || payload.ROOT_FOLDER_ID || payload.folderId || payload.FOLDER_ID || 0);
            state.currentFolderId = Number(payload.currentFolderId || payload.CURRENT_FOLDER_ID || payload.folderId || payload.FOLDER_ID || state.rootFolderId || 0);

            var rawItems = payload.items || payload.files || payload.children || [];
            state.items = Array.isArray(rawItems) ? rawItems.map(normalizeItem) : [];

            state.breadcrumbs = payload.breadcrumbs || payload.path || [];
            state.permissions = payload.permissions || payload.rights || {};

            setLoading(state, false);
            render(state);
        }).catch(function (err) {
            setLoading(state, false);
            setMessage(state, 'Ошибка загрузки диска: ' + (err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR'), 'error');
            render(state);
        });
    }

    function loadFolder(state, folderId) {
        folderId = Number(folderId || 0);

        setLoading(state, true);
        setMessage(state, '', '');

        return api(state.root, 'loadFolder', {
            folderId: folderId
        }).then(function (res) {
            var payload = getPayload(res);

            state.currentFolderId = Number(payload.currentFolderId || payload.CURRENT_FOLDER_ID || payload.folderId || payload.FOLDER_ID || folderId || 0);

            var rawItems = payload.items || payload.files || payload.children || [];
            state.items = Array.isArray(rawItems) ? rawItems.map(normalizeItem) : [];

            state.breadcrumbs = payload.breadcrumbs || payload.path || [];

            setLoading(state, false);
            render(state);
        }).catch(function (err) {
            setLoading(state, false);
            setMessage(state, 'Ошибка открытия папки: ' + (err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR'), 'error');
            render(state);
        });
    }

    function findItem(state, id) {
        id = Number(id || 0);

        for (var i = 0; i < state.items.length; i++) {
            if (Number(state.items[i].id) === id) {
                return state.items[i];
            }
        }

        return null;
    }

    function openFile(state, item) {
        if (!item) return;

        if (item.viewUrl) {
            window.open(item.viewUrl, '_blank');
            return;
        }

        if (item.downloadUrl) {
            window.open(item.downloadUrl, '_blank');
            return;
        }

        downloadItem(state, item);
    }

    function downloadItem(state, item) {
        if (!item) return;

        api(state.root, 'download', {
            objectId: item.id,
            id: item.id
        }).then(function (res) {
            var payload = getPayload(res);
            var url = payload.url || payload.downloadUrl || payload.DOWNLOAD_URL || '';

            if (url) {
                window.open(url, '_blank');
            } else {
                setMessage(state, 'Ссылка на скачивание не получена', 'error');
            }
        }).catch(function (err) {
            setMessage(state, 'Ошибка скачивания: ' + (err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR'), 'error');
        });
    }

    function uploadFiles(state, files) {
        files = Array.prototype.slice.call(files || []);

        if (!files.length) {
            return;
        }

        setMessage(state, 'Загружаю файлов: ' + files.length, '');

        var chain = Promise.resolve();

        files.forEach(function (file) {
            chain = chain.then(function () {
                return apiUpload(state.root, state.currentFolderId, file);
            });
        });

        chain.then(function () {
            setMessage(state, 'Файлы загружены', 'success');
            return loadFolder(state, state.currentFolderId);
        }).catch(function (err) {
            setMessage(state, 'Ошибка загрузки: ' + (err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR'), 'error');
        });
    }

    function createFolder(state) {
        var name = prompt('Название новой папки');

        if (name === null) return;

        name = String(name || '').trim();

        if (!name) {
            alert('Введите название папки');
            return;
        }

        api(state.root, 'createFolder', {
            folderId: state.currentFolderId,
            name: name
        }).then(function () {
            setMessage(state, 'Папка создана', 'success');
            return loadFolder(state, state.currentFolderId);
        }).catch(function (err) {
            setMessage(state, 'Ошибка создания папки: ' + (err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR'), 'error');
        });
    }

    function renameItem(state, item) {
        if (!item) return;

        var name = prompt('Новое название', item.name);

        if (name === null) return;

        name = String(name || '').trim();

        if (!name) {
            alert('Введите новое название');
            return;
        }

        api(state.root, 'rename', {
            objectId: item.id,
            id: item.id,
            name: name
        }).then(function () {
            setMessage(state, 'Название изменено', 'success');
            return loadFolder(state, state.currentFolderId);
        }).catch(function (err) {
            setMessage(state, 'Ошибка переименования: ' + (err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR'), 'error');
        });
    }

    function deleteItem(state, item) {
        if (!item) return;

        if (!confirm('Удалить "' + item.name + '"?')) {
            return;
        }

        api(state.root, 'delete', {
            objectId: item.id,
            id: item.id
        }).then(function () {
            setMessage(state, 'Удалено', 'success');
            return loadFolder(state, state.currentFolderId);
        }).catch(function (err) {
            setMessage(state, 'Ошибка удаления: ' + (err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR'), 'error');
        });
    }

    function bindEvents(state) {
        state.root.addEventListener('click', function (e) {
            var button = e.target.closest('button');

            if (!button || !state.root.contains(button)) {
                return;
            }

            var action = button.getAttribute('data-action') || '';
            var itemId = Number(button.getAttribute('data-item-id') || 0);
            var item = itemId ? findItem(state, itemId) : null;

            if (action === 'refresh') {
                loadFolder(state, state.currentFolderId);
                return;
            }

            if (action === 'upload') {
                var input = state.root.querySelector('[data-role="file-input"]');

                if (input) {
                    input.click();
                }

                return;
            }

            if (action === 'create-folder') {
                createFolder(state);
                return;
            }

            if (action === 'view-table') {
                state.view = 'table';
                render(state);
                return;
            }

            if (action === 'view-grid') {
                state.view = 'grid';
                render(state);
                return;
            }

            if (action === 'open-folder' && item) {
                loadFolder(state, item.id);
                return;
            }

            if (action === 'open-file' && item) {
                openFile(state, item);
                return;
            }

            if (action === 'download' && item) {
                downloadItem(state, item);
                return;
            }

            if (action === 'rename' && item) {
                renameItem(state, item);
                return;
            }

            if (action === 'delete' && item) {
                deleteItem(state, item);
                return;
            }

            var folderId = Number(button.getAttribute('data-folder-id') || 0);

            if (folderId > 0) {
                loadFolder(state, folderId);
            }
        });

        var search = state.root.querySelector('[data-role="search"]');
        if (search) {
            search.addEventListener('input', function () {
                state.query = search.value || '';
                render(state);
            });
        }

        var sort = state.root.querySelector('[data-role="sort"]');
        if (sort) {
            sort.addEventListener('change', function () {
                state.sort = sort.value || 'new';
                render(state);
            });
        }

        var fileInput = state.root.querySelector('[data-role="file-input"]');
        if (fileInput) {
            fileInput.addEventListener('change', function () {
                uploadFiles(state, fileInput.files);
                fileInput.value = '';
            });
        }
    }

    function initDisk(root) {
        if (!root || root.getAttribute('data-modern-disk-ready') === '1') {
            return;
        }

        root.setAttribute('data-modern-disk-ready', '1');
        root.classList.add('sb-disk-enhanced');

        buildApp(root);

        var state = createState(root);

        bindEvents(state);
        loadBootstrap(state);
    }

    function initAll() {
        document.querySelectorAll('.sb-disk').forEach(initDisk);
    }

    document.addEventListener('DOMContentLoaded', initAll);

    if (document.readyState === 'interactive' || document.readyState === 'complete') {
        initAll();
    }
})();

И не забудь в public_page.php подключение оставить с новой версией:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=4"></script>

После замены обнови страницу через Ctrl + F5.