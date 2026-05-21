Ошибка из-за моего нового script.js: я поставил загрузку с действием:

action = 'upload'

А твой components/disk/api.php, судя по ответу, такое действие не знает. Сделаем безопасно: при загрузке файл будет пробовать несколько вариантов действия, пока не найдёт тот, который понимает твой API.

В файле:

/local/sitebuilder/components/disk/script.js

найди функцию:

function apiUpload(root, folderId, file) {

и замени её целиком на эту:

function apiUpload(root, folderId, file) {
    var uploadActions = [
        'file.upload',
        'uploadFile',
        'disk.upload',
        'upload'
    ];

    function buildFormData(actionName) {
        var fd = new FormData();

        fd.append('action', actionName);
        fd.append('sessid', getSessid(root));
        fd.append('siteId', root.getAttribute('data-site-id') || '0');
        fd.append('pageId', root.getAttribute('data-page-id') || '0');
        fd.append('blockId', root.getAttribute('data-block-id') || '0');

        fd.append('folderId', String(folderId || 0));
        fd.append('currentFolderId', String(folderId || 0));
        fd.append('parentId', String(folderId || 0));

        fd.append('file', file);

        return fd;
    }

    function sendUpload(actionName) {
        return fetch(API_URL, {
            method: 'POST',
            credentials: 'same-origin',
            body: buildFormData(actionName)
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

    function tryAction(index, lastError) {
        if (index >= uploadActions.length) {
            throw lastError || {
                ok: false,
                error: 'UPLOAD_ACTION_NOT_FOUND'
            };
        }

        return sendUpload(uploadActions[index]).catch(function (err) {
            if (err && err.error === 'UNKNOWN_ACTION') {
                return tryAction(index + 1, err);
            }

            throw err;
        });
    }

    return tryAction(0, null);
}

Потом в public_page.php обнови версию скрипта:

<script src="<?= sb_public_h($basePath) ?>/components/disk/script.js?v=5"></script>

После этого сделай Ctrl + F5 и попробуй загрузить файл.

Если следующая ошибка будет уже не UNKNOWN_ACTION, а например FILE_REQUIRED или BAD_FOLDER_ID, значит действие мы нашли, и останется подстроить имя поля или ID папки.