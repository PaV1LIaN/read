Отлично. Дальше делаем UI настроек блока “Диск” так, чтобы это уже было похоже на реальный компонент конструктора.

Ниже дам:

HTML модалки настроек;

CSS для нее;

JS открытия/закрытия/загрузки/сохранения;

обновленный template.php;

action сохранения уже под этот UI;

action получения списка доступных root-вариантов;

рекомендации, как встроить это в ваш editor.



---

1. Что получится после этого шага

После этого пакета у блока уже будет рабочая панель настроек, где можно менять:

заголовок блока;

root папку блока;

использовать ли корень сайта;

режим отображения по умолчанию;

разрешение на загрузку;

разрешение на создание папок;

разрешение на переименование;

разрешение на удаление;

разрешение на скачивание;

показывать ли поиск;

показывать ли breadcrumbs;

сортировку;

лимит файла;

допустимые расширения;

режим прав.



---

2. Новый action: список root-вариантов для настроек

Чтобы в модалке можно было выбирать “корень сайта” или “собственную папку блока”, сделаем action.

actions/get_root_options.php

<?php

DiskCsrf::validateFromRequest();
$data = disk_read_json_body();

$currentUserId = DiskCurrentUser::requireId();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => $currentUserId,
]);

DiskValidator::assertContext($context);

$settings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$permissions = DiskPermissionService::resolve($context, $settings, null);
DiskValidator::assertCan($permissions, 'canEditSettings');

$site = SiteRepository::getById($context->siteId);
$siteRootFolderId = SiteRepository::getRootDiskFolderId($context->siteId);

$options = [];

if ($siteRootFolderId) {
    $options[] = [
        'value' => '',
        'label' => 'Использовать корень сайта',
        'type' => 'site_root',
        'folderId' => (int)$siteRootFolderId,
    ];
}

if (!empty($settings['rootFolderId'])) {
    $options[] = [
        'value' => (int)$settings['rootFolderId'],
        'label' => 'Собственная папка блока',
        'type' => 'block_root',
        'folderId' => (int)$settings['rootFolderId'],
    ];
}

DiskResponse::success([
    'options' => $options,
    'siteRootFolderId' => $siteRootFolderId ? (int)$siteRootFolderId : null,
    'blockRootFolderId' => !empty($settings['rootFolderId']) ? (int)$settings['rootFolderId'] : null,
    'siteName' => $site ? (string)$site['name'] : '',
]);


---

3. Обновленный api.php

Добавляем новый action:

<?php

require_once __DIR__ . '/bootstrap.php';

try {
    $action = $_GET['action'] ?? '';

    switch ($action) {
        case 'resolveRoot':
            require __DIR__ . '/actions/resolve_root.php';
            break;

        case 'getSettings':
            require __DIR__ . '/actions/get_settings.php';
            break;

        case 'saveSettings':
            require __DIR__ . '/actions/save_settings.php';
            break;

        case 'getPermissions':
            require __DIR__ . '/actions/get_permissions.php';
            break;

        case 'getRootOptions':
            require __DIR__ . '/actions/get_root_options.php';
            break;

        case 'list':
            require __DIR__ . '/actions/list.php';
            break;

        case 'upload':
            require __DIR__ . '/actions/upload.php';
            break;

        case 'createFolder':
            require __DIR__ . '/actions/create_folder.php';
            break;

        case 'rename':
            require __DIR__ . '/actions/rename.php';
            break;

        case 'delete':
            require __DIR__ . '/actions/delete.php';
            break;

        case 'move':
            require __DIR__ . '/actions/move.php';
            break;

        case 'copy':
            require __DIR__ . '/actions/copy.php';
            break;

        case 'search':
            require __DIR__ . '/actions/search.php';
            break;

        case 'download':
            require __DIR__ . '/actions/download.php';
            break;

        case 'initSiteRoot':
            require __DIR__ . '/actions/init_site_root.php';
            break;

        case 'initBlockRoot':
            require __DIR__ . '/actions/init_block_root.php';
            break;

        default:
            DiskResponse::error('UNKNOWN_ACTION', 'Неизвестное действие');
    }
} catch (Throwable $e) {
    DiskResponse::error('SERVER_ERROR', $e->getMessage());
}


---

4. Обновленный actions/save_settings.php

Ниже версия, полностью совместимая с UI формы.

actions/save_settings.php

<?php

DiskCsrf::validateFromRequest();
$data = disk_read_json_body();

$currentUserId = DiskCurrentUser::requireId();

$context = DiskContextFactory::fromArray([
    'siteId' => (int)($data['siteId'] ?? 0),
    'pageId' => (int)($data['pageId'] ?? 0),
    'blockId' => (int)($data['blockId'] ?? 0),
    'currentUserId' => $currentUserId,
]);

DiskValidator::assertContext($context);

$currentSettings = DiskSettingsRepository::ensureExistsForBlock(
    $context->blockId,
    $context->siteId,
    $context->pageId,
    $context->currentUserId
);

$rootFolderId = DiskRootResolver::resolve($context, $currentSettings);
$permissions = DiskPermissionService::resolve($context, $currentSettings, $rootFolderId);

DiskValidator::assertCan($permissions, 'canEditSettings');

$settings = $data['settings'] ?? [];
if (!is_array($settings)) {
    throw new RuntimeException('INVALID_SETTINGS_PAYLOAD');
}

$allowedExtensions = [];
if (isset($settings['allowedExtensions'])) {
    if (is_array($settings['allowedExtensions'])) {
        $allowedExtensions = $settings['allowedExtensions'];
    } else {
        $raw = trim((string)$settings['allowedExtensions']);
        if ($raw !== '') {
            $allowedExtensions = preg_split('/[\s,;]+/u', $raw);
        }
    }
}

$rootFolderIdValue = null;
if (array_key_exists('rootFolderId', $settings)) {
    $tmp = trim((string)$settings['rootFolderId']);
    $rootFolderIdValue = ($tmp !== '' && (int)$tmp > 0) ? (int)$tmp : null;
}

$normalized = [
    'title' => trim((string)($settings['title'] ?? 'Файлы')),
    'rootFolderId' => $rootFolderIdValue,
    'viewMode' => in_array((string)($settings['viewMode'] ?? 'table'), ['table', 'grid'], true)
        ? (string)$settings['viewMode']
        : 'table',
    'allowUpload' => disk_normalize_bool($settings['allowUpload'] ?? true),
    'allowCreateFolder' => disk_normalize_bool($settings['allowCreateFolder'] ?? true),
    'allowRename' => disk_normalize_bool($settings['allowRename'] ?? true),
    'allowDelete' => disk_normalize_bool($settings['allowDelete'] ?? false),
    'allowDownload' => disk_normalize_bool($settings['allowDownload'] ?? true),
    'showSearch' => disk_normalize_bool($settings['showSearch'] ?? true),
    'showBreadcrumbs' => disk_normalize_bool($settings['showBreadcrumbs'] ?? true),
    'defaultSort' => trim((string)($settings['defaultSort'] ?? 'updatedAt')),
    'defaultSortDirection' => strtolower((string)($settings['defaultSortDirection'] ?? 'desc')) === 'asc' ? 'asc' : 'desc',
    'allowedExtensions' => array_values(array_filter(array_map(static function ($value) {
        return strtolower(trim((string)$value));
    }, $allowedExtensions))),
    'maxFileSize' => max(0, (int)($settings['maxFileSize'] ?? 52428800)),
    'permissionMode' => in_array((string)($settings['permissionMode'] ?? 'inherit_site'), ['inherit_site', 'custom'], true)
        ? (string)$settings['permissionMode']
        : 'inherit_site',
    'useSiteRootFallback' => disk_normalize_bool($settings['useSiteRootFallback'] ?? true),
];

DiskSettingsRepository::save($context->blockId, $normalized);

$updatedSettings = DiskSettingsRepository::getByBlockId($context->blockId);

DiskResponse::success([
    'settings' => $updatedSettings,
]);


---

5. Модалка настроек блока

Добавим HTML модалки прямо в template.php.


---

Обновленный template.php

<?php
$initialStateJson = disk_h(json_encode($arResult['INITIAL_STATE'], JSON_UNESCAPED_UNICODE));
?>
<div class="sb-disk"
     id="sb-disk-<?= (int)$arResult['BLOCK_ID'] ?>"
     data-site-id="<?= (int)$arResult['SITE_ID'] ?>"
     data-page-id="<?= (int)$arResult['PAGE_ID'] ?>"
     data-block-id="<?= (int)$arResult['BLOCK_ID'] ?>"
     data-initial-state="<?= $initialStateJson ?>">

    <div class="sb-disk__header">
        <div class="sb-disk__header-main">
            <div class="sb-disk__title-wrap">
                <h3 class="sb-disk__title"><?= disk_h($arResult['TITLE']) ?></h3>
                <div class="sb-disk__subtitle" data-role="subtitle"></div>
            </div>
        </div>

        <div class="sb-disk__header-actions">
            <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="refresh">Обновить</button>

            <?php if (!empty($arResult['PERMISSIONS']['canEditSettings'])): ?>
                <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="settings">Настройки</button>
            <?php endif; ?>
        </div>
    </div>

    <?php if (!empty($arResult['SETTINGS']['showBreadcrumbs'])): ?>
        <div class="sb-disk__breadcrumbs" data-role="breadcrumbs"></div>
    <?php endif; ?>

    <div class="sb-disk__toolbar">
        <div class="sb-disk__toolbar-left">
            <?php if (!empty($arResult['SETTINGS']['showSearch'])): ?>
                <div class="sb-disk__search">
                    <input type="text"
                           class="sb-disk__search-input"
                           data-role="search-input"
                           placeholder="Поиск файлов и папок">
                </div>
            <?php endif; ?>

            <select class="sb-disk__select" data-role="sort-select">
                <option value="updatedAt:desc">Сначала новые</option>
                <option value="updatedAt:asc">Сначала старые</option>
                <option value="name:asc">По имени А–Я</option>
                <option value="name:desc">По имени Я–А</option>
                <option value="size:desc">По размеру</option>
            </select>
        </div>

        <div class="sb-disk__toolbar-right">
            <?php if (!empty($arResult['PERMISSIONS']['canUpload'])): ?>
                <button type="button" class="sb-disk__btn" data-action="upload">Загрузить</button>
            <?php endif; ?>

            <?php if (!empty($arResult['PERMISSIONS']['canCreateFolder'])): ?>
                <button type="button" class="sb-disk__btn" data-action="create-folder">Новая папка</button>
            <?php endif; ?>

            <div class="sb-disk__view-switch">
                <button type="button" class="sb-disk__view-btn is-active" data-view="table">Таблица</button>
                <button type="button" class="sb-disk__view-btn" data-view="grid">Плитка</button>
            </div>
        </div>
    </div>

    <div class="sb-disk__bulkbar" data-role="bulkbar" hidden>
        <span class="sb-disk__bulkbar-text" data-role="bulkbar-text">Выбрано: 0</span>
        <div class="sb-disk__bulkbar-actions">
            <button type="button" class="sb-disk__btn" data-action="download-selected">Скачать</button>
            <button type="button" class="sb-disk__btn sb-disk__btn--danger" data-action="delete-selected">Удалить</button>
        </div>
    </div>

    <div class="sb-disk__content">
        <div class="sb-disk__state" data-state="loading" hidden>Загрузка...</div>
        <div class="sb-disk__state" data-state="empty" hidden>Здесь пока нет файлов и папок.</div>
        <div class="sb-disk__state" data-state="error" hidden>Не удалось загрузить содержимое.</div>
        <div class="sb-disk__state" data-state="no-access" hidden>У вас нет доступа к этому разделу.</div>
        <div class="sb-disk__state" data-state="no-root" hidden>
            Для блока не настроена корневая папка.
            <?php if (!empty($arResult['PERMISSIONS']['canEditSettings'])): ?>
                <div class="sb-disk__state-actions">
                    <button type="button" class="sb-disk__btn" data-action="init-site-root">Создать корень сайта</button>
                    <button type="button" class="sb-disk__btn" data-action="init-block-root">Создать папку блока</button>
                </div>
            <?php endif; ?>
        </div>

        <div class="sb-disk__view sb-disk__view--table" data-view-container="table">
            <table class="sb-disk__table">
                <thead>
                    <tr>
                        <th class="sb-disk__col sb-disk__col--checkbox">
                            <input type="checkbox" data-role="select-all">
                        </th>
                        <th class="sb-disk__col sb-disk__col--name">Название</th>
                        <th class="sb-disk__col">Тип</th>
                        <th class="sb-disk__col">Размер</th>
                        <th class="sb-disk__col">Изменен</th>
                        <th class="sb-disk__col sb-disk__col--actions"></th>
                    </tr>
                </thead>
                <tbody data-role="items-table"></tbody>
            </table>
        </div>

        <div class="sb-disk__view sb-disk__view--grid" data-view-container="grid" hidden></div>
    </div>

    <input type="file" class="sb-disk__file-input" data-role="upload-input" multiple hidden>

    <?php if (!empty($arResult['PERMISSIONS']['canEditSettings'])): ?>
        <div class="sb-disk-modal" data-role="settings-modal" hidden>
            <div class="sb-disk-modal__backdrop" data-action="close-settings"></div>
            <div class="sb-disk-modal__dialog">
                <div class="sb-disk-modal__header">
                    <h3 class="sb-disk-modal__title">Настройки блока “Диск”</h3>
                    <button type="button" class="sb-disk-modal__close" data-action="close-settings">×</button>
                </div>

                <div class="sb-disk-modal__body">
                    <form class="sb-disk-form" data-role="settings-form">
                        <div class="sb-disk-form__grid">
                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Заголовок блока</label>
                                <input type="text" class="sb-disk-form__input" name="title">
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Источник корня</label>
                                <select class="sb-disk-form__select" name="rootFolderId" data-role="root-select">
                                    <option value="">Использовать корень сайта</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Вид по умолчанию</label>
                                <select class="sb-disk-form__select" name="viewMode">
                                    <option value="table">Таблица</option>
                                    <option value="grid">Плитка</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Сортировка по умолчанию</label>
                                <select class="sb-disk-form__select" name="defaultSort">
                                    <option value="updatedAt">Дата изменения</option>
                                    <option value="createdAt">Дата создания</option>
                                    <option value="name">Имя</option>
                                    <option value="size">Размер</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Направление сортировки</label>
                                <select class="sb-disk-form__select" name="defaultSortDirection">
                                    <option value="desc">По убыванию</option>
                                    <option value="asc">По возрастанию</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Максимальный размер файла (байт)</label>
                                <input type="number" class="sb-disk-form__input" name="maxFileSize" min="0">
                            </div>

                            <div class="sb-disk-form__field sb-disk-form__field--full">
                                <label class="sb-disk-form__label">Допустимые расширения</label>
                                <input type="text" class="sb-disk-form__input" name="allowedExtensions" placeholder="pdf doc docx xlsx png jpg">
                                <div class="sb-disk-form__hint">Через пробел, запятую или точку с запятой</div>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__label">Режим прав</label>
                                <select class="sb-disk-form__select" name="permissionMode">
                                    <option value="inherit_site">Наследовать права сайта</option>
                                    <option value="custom">Собственные ограничения блока</option>
                                </select>
                            </div>

                            <div class="sb-disk-form__field">
                                <label class="sb-disk-form__check">
                                    <input type="checkbox" name="useSiteRootFallback" value="1">
                                    <span>Использовать корень сайта, если у блока нет своей папки</span>
                                </label>
                            </div>
                        </div>

                        <div class="sb-disk-form__checks">
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowUpload" value="1"><span>Разрешить загрузку</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowCreateFolder" value="1"><span>Разрешить создание папок</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowRename" value="1"><span>Разрешить переименование</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowDelete" value="1"><span>Разрешить удаление</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="allowDownload" value="1"><span>Разрешить скачивание</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="showSearch" value="1"><span>Показывать поиск</span></label>
                            <label class="sb-disk-form__check"><input type="checkbox" name="showBreadcrumbs" value="1"><span>Показывать breadcrumbs</span></label>
                        </div>
                    </form>

                    <div class="sb-disk-modal__message" data-role="settings-message"></div>
                </div>

                <div class="sb-disk-modal__footer">
                    <button type="button" class="sb-disk__btn sb-disk__btn--ghost" data-action="close-settings">Отмена</button>
                    <button type="button" class="sb-disk__btn" data-action="save-settings">Сохранить</button>
                </div>
            </div>
        </div>
    <?php endif; ?>
</div>


---

6. CSS для модалки и формы настроек

Добавьте в styles.css.

Дополнение к styles.css

.sb-disk__state-actions {
    margin-top: 14px;
    display: flex;
    justify-content: center;
    gap: 10px;
    flex-wrap: wrap;
}

.sb-disk-modal[hidden] {
    display: none !important;
}

.sb-disk-modal {
    position: fixed;
    inset: 0;
    z-index: 9999;
}

.sb-disk-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(17, 24, 39, 0.45);
}

.sb-disk-modal__dialog {
    position: relative;
    z-index: 2;
    width: min(920px, calc(100vw - 32px));
    max-height: calc(100vh - 32px);
    overflow: auto;
    margin: 16px auto;
    background: #fff;
    border-radius: 18px;
    box-shadow: 0 20px 60px rgba(0, 0, 0, 0.18);
}

.sb-disk-modal__header,
.sb-disk-modal__footer {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 12px;
    padding: 18px 20px;
    border-bottom: 1px solid #eef2f7;
}

.sb-disk-modal__footer {
    border-top: 1px solid #eef2f7;
    border-bottom: none;
}

.sb-disk-modal__title {
    margin: 0;
    font-size: 20px;
    line-height: 1.2;
    font-weight: 600;
}

.sb-disk-modal__close {
    appearance: none;
    border: none;
    background: transparent;
    font-size: 28px;
    line-height: 1;
    cursor: pointer;
}

.sb-disk-modal__body {
    padding: 20px;
    display: flex;
    flex-direction: column;
    gap: 18px;
}

.sb-disk-modal__message {
    min-height: 20px;
    font-size: 14px;
    color: #6b7280;
}

.sb-disk-form {
    display: flex;
    flex-direction: column;
    gap: 20px;
}

.sb-disk-form__grid {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 16px;
}

.sb-disk-form__field {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-disk-form__field--full {
    grid-column: 1 / -1;
}

.sb-disk-form__label {
    font-size: 14px;
    font-weight: 500;
    color: #111827;
}

.sb-disk-form__input,
.sb-disk-form__select {
    height: 42px;
    border: 1px solid #d1d5db;
    border-radius: 10px;
    background: #fff;
    padding: 0 12px;
    font-size: 14px;
    box-sizing: border-box;
}

.sb-disk-form__hint {
    font-size: 12px;
    color: #6b7280;
}

.sb-disk-form__checks {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 12px 16px;
}

.sb-disk-form__check {
    display: inline-flex;
    align-items: center;
    gap: 10px;
    font-size: 14px;
    color: #111827;
}

@media (max-width: 760px) {
    .sb-disk-form__grid,
    .sb-disk-form__checks {
        grid-template-columns: 1fr;
    }
}


---

7. JS для модалки настроек

Теперь обновим script.js, добавив:

открытие модалки;

загрузку getSettings;

загрузку getRootOptions;

заполнение формы;

сохранение через saveSettings;

инициализацию root сайта/блока.


Ниже добавочный блок методов, который надо встроить в ваш DiskComponent.


---

Дополнение к script.js

Добавить в bindStaticEvents()

var settingsBtn = this.root.querySelector('[data-action="settings"]');
if (settingsBtn) {
  settingsBtn.addEventListener('click', async function () {
    await self.openSettingsModal();
  });
}

var closeSettingsBtns = this.root.querySelectorAll('[data-action="close-settings"]');
closeSettingsBtns.forEach(function (btn) {
  btn.addEventListener('click', function () {
    self.closeSettingsModal();
  });
});

var saveSettingsBtn = this.root.querySelector('[data-action="save-settings"]');
if (saveSettingsBtn) {
  saveSettingsBtn.addEventListener('click', async function () {
    await self.saveSettings();
  });
}

var initSiteRootBtn = this.root.querySelector('[data-action="init-site-root"]');
if (initSiteRootBtn) {
  initSiteRootBtn.addEventListener('click', async function () {
    await self.initSiteRoot();
  });
}

var initBlockRootBtn = this.root.querySelector('[data-action="init-block-root"]');
if (initBlockRootBtn) {
  initBlockRootBtn.addEventListener('click', async function () {
    await self.initBlockRoot();
  });
}


---

Добавить методы в DiskComponent.prototype

DiskComponent.prototype.openSettingsModal = async function () {
  var modal = this.root.querySelector('[data-role="settings-modal"]');
  if (!modal) return;

  modal.hidden = false;
  this.setSettingsMessage('Загрузка настроек...');

  try {
    var settingsPayload = this.getBasePayload();
    settingsPayload.sessid = this.getSessid();

    var settingsRes = await this.api('getSettings', settingsPayload);
    if (!settingsRes.ok) {
      throw new Error(settingsRes.message || settingsRes.error || 'GET_SETTINGS_ERROR');
    }

    var rootsPayload = this.getBasePayload();
    rootsPayload.sessid = this.getSessid();

    var rootOptionsRes = await this.api('getRootOptions', rootsPayload);
    if (!rootOptionsRes.ok) {
      throw new Error(rootOptionsRes.message || rootOptionsRes.error || 'GET_ROOT_OPTIONS_ERROR');
    }

    this.fillSettingsForm(
      settingsRes.data.settings || {},
      rootOptionsRes.data || {}
    );

    this.setSettingsMessage('');
  } catch (e) {
    console.error(e);
    this.setSettingsMessage('Не удалось загрузить настройки.');
  }
};

DiskComponent.prototype.closeSettingsModal = function () {
  var modal = this.root.querySelector('[data-role="settings-modal"]');
  if (!modal) return;

  modal.hidden = true;
};

DiskComponent.prototype.getSessid = function () {
  if (window.BX && typeof BX.bitrix_sessid === 'function') {
    return BX.bitrix_sessid();
  }
  return '';
};

DiskComponent.prototype.setSettingsMessage = function (message) {
  var node = this.root.querySelector('[data-role="settings-message"]');
  if (!node) return;

  node.textContent = message || '';
};

DiskComponent.prototype.fillSettingsForm = function (settings, rootData) {
  var form = this.root.querySelector('[data-role="settings-form"]');
  if (!form) return;

  var rootSelect = form.querySelector('[data-role="root-select"]');
  if (rootSelect) {
    var options = Array.isArray(rootData.options) ? rootData.options : [];
    rootSelect.innerHTML = '';

    if (rootData.siteRootFolderId) {
      rootSelect.insertAdjacentHTML('beforeend',
        '<option value="">Использовать корень сайта</option>'
      );
    } else {
      rootSelect.insertAdjacentHTML('beforeend',
        '<option value="">Корень сайта не создан</option>'
      );
    }

    options.forEach(function (option) {
      if (option.type === 'block_root' && option.folderId) {
        rootSelect.insertAdjacentHTML('beforeend',
          '<option value="' + escapeHtml(option.folderId) + '">Собственная папка блока #' + escapeHtml(option.folderId) + '</option>'
        );
      }
    });
  }

  setFormValue(form, 'title', settings.title || 'Файлы');
  setFormValue(form, 'rootFolderId', settings.rootFolderId || '');
  setFormValue(form, 'viewMode', settings.viewMode || 'table');
  setFormValue(form, 'defaultSort', settings.defaultSort || 'updatedAt');
  setFormValue(form, 'defaultSortDirection', settings.defaultSortDirection || 'desc');
  setFormValue(form, 'maxFileSize', settings.maxFileSize || 52428800);
  setFormValue(form, 'permissionMode', settings.permissionMode || 'inherit_site');

  var extValue = Array.isArray(settings.allowedExtensions)
    ? settings.allowedExtensions.join(' ')
    : '';
  setFormValue(form, 'allowedExtensions', extValue);

  setFormCheckbox(form, 'allowUpload', !!settings.allowUpload);
  setFormCheckbox(form, 'allowCreateFolder', !!settings.allowCreateFolder);
  setFormCheckbox(form, 'allowRename', !!settings.allowRename);
  setFormCheckbox(form, 'allowDelete', !!settings.allowDelete);
  setFormCheckbox(form, 'allowDownload', !!settings.allowDownload);
  setFormCheckbox(form, 'showSearch', !!settings.showSearch);
  setFormCheckbox(form, 'showBreadcrumbs', !!settings.showBreadcrumbs);
  setFormCheckbox(form, 'useSiteRootFallback', !!settings.useSiteRootFallback);
};

DiskComponent.prototype.collectSettingsForm = function () {
  var form = this.root.querySelector('[data-role="settings-form"]');
  if (!form) return {};

  return {
    title: getFormValue(form, 'title'),
    rootFolderId: getFormValue(form, 'rootFolderId'),
    viewMode: getFormValue(form, 'viewMode'),
    defaultSort: getFormValue(form, 'defaultSort'),
    defaultSortDirection: getFormValue(form, 'defaultSortDirection'),
    maxFileSize: Number(getFormValue(form, 'maxFileSize') || 0),
    allowedExtensions: getFormValue(form, 'allowedExtensions'),
    permissionMode: getFormValue(form, 'permissionMode'),
    allowUpload: getFormCheckbox(form, 'allowUpload'),
    allowCreateFolder: getFormCheckbox(form, 'allowCreateFolder'),
    allowRename: getFormCheckbox(form, 'allowRename'),
    allowDelete: getFormCheckbox(form, 'allowDelete'),
    allowDownload: getFormCheckbox(form, 'allowDownload'),
    showSearch: getFormCheckbox(form, 'showSearch'),
    showBreadcrumbs: getFormCheckbox(form, 'showBreadcrumbs'),
    useSiteRootFallback: getFormCheckbox(form, 'useSiteRootFallback')
  };
};

DiskComponent.prototype.saveSettings = async function () {
  try {
    this.setSettingsMessage('Сохранение...');

    var payload = this.getBasePayload();
    payload.sessid = this.getSessid();
    payload.settings = this.collectSettingsForm();

    var res = await this.api('saveSettings', payload);
    if (!res.ok) {
      throw new Error(res.message || res.error || 'SAVE_SETTINGS_ERROR');
    }

    this.state.settings = res.data.settings || this.state.settings;
    this.state.viewMode = this.state.settings.viewMode || 'table';
    this.applyInitialViewMode();

    this.setSettingsMessage('Настройки сохранены.');
    this.closeSettingsModal();

    var rootResPayload = this.getBasePayload();
    rootResPayload.sessid = this.getSessid();

    var rootRes = await this.api('resolveRoot', rootResPayload);
    if (rootRes.ok) {
      this.state.rootFolderId = rootRes.data ? rootRes.data.rootFolderId : rootRes.rootFolderId;
      this.state.currentFolderId = this.state.rootFolderId;
    }

    if (this.state.rootFolderId) {
      await this.loadFolder(this.state.rootFolderId);
    } else {
      this.renderState('no-root');
    }
  } catch (e) {
    console.error(e);
    this.setSettingsMessage('Не удалось сохранить настройки.');
  }
};

DiskComponent.prototype.initSiteRoot = async function () {
  try {
    var payload = {
      siteId: this.state.siteId,
      sessid: this.getSessid()
    };

    var res = await this.api('initSiteRoot', payload);
    if (!res.ok) {
      throw new Error(res.message || res.error || 'INIT_SITE_ROOT_ERROR');
    }

    await this.loadResolvedRoot();
  } catch (e) {
    console.error(e);
    alert('Не удалось создать корень сайта');
  }
};

DiskComponent.prototype.initBlockRoot = async function () {
  try {
    var payload = this.getBasePayload();
    payload.sessid = this.getSessid();

    var res = await this.api('initBlockRoot', payload);
    if (!res.ok) {
      throw new Error(res.message || res.error || 'INIT_BLOCK_ROOT_ERROR');
    }

    await this.loadResolvedRoot();
  } catch (e) {
    console.error(e);
    alert('Не удалось создать папку блока');
  }
};

DiskComponent.prototype.loadResolvedRoot = async function () {
  var payload = this.getBasePayload();
  payload.sessid = this.getSessid();

  var rootRes = await this.api('resolveRoot', payload);
  if (!rootRes.ok) {
    throw new Error(rootRes.message || rootRes.error || 'RESOLVE_ROOT_ERROR');
  }

  var data = rootRes.data || rootRes;
  this.state.rootFolderId = data.rootFolderId || null;
  this.state.currentFolderId = this.state.rootFolderId || null;

  if (this.state.rootFolderId) {
    await this.loadFolder(this.state.rootFolderId);
  } else {
    this.renderState('no-root');
  }
};


---

Внизу файла добавить helper-функции

function setFormValue(form, name, value) {
  var node = form.querySelector('[name="' + name + '"]');
  if (!node) return;
  node.value = value == null ? '' : value;
}

function getFormValue(form, name) {
  var node = form.querySelector('[name="' + name + '"]');
  return node ? String(node.value || '') : '';
}

function setFormCheckbox(form, name, checked) {
  var node = form.querySelector('[name="' + name + '"]');
  if (!node) return;
  node.checked = !!checked;
}

function getFormCheckbox(form, name) {
  var node = form.querySelector('[name="' + name + '"]');
  return !!(node && node.checked);
}


---

8. Важная правка метода api() в script.js

Чтобы JSON-запросы всегда отправляли sessid, ничего менять в самом методе не надо, если вы его уже вручную прокидываете в payload.

Но для upload лучше тоже добавить sessid.

Найдите в upload:

formData.append('currentFolderId', self.state.currentFolderId);

и добавьте:

formData.append('sessid', self.getSessid());


---

9. Правка серверных upload/list/... action для sessid

Теперь все write-action должны начинаться с:

DiskCsrf::validateFromRequest();

Это обязательно для:

saveSettings

create_folder

upload

rename

delete

move

copy

initSiteRoot

initBlockRoot


Для getSettings и list тоже можно оставить валидацию, это нормально для вашего проекта.


---

10. Что лучше еще чуть поправить в JS

Есть нюанс: в resolveRoot у вас ответ идет через DiskResponse::success, значит данные лежат в res.data.

Лучше сделать единый helper:

function getApiData(res) {
  return res && res.data ? res.data : {};
}

И использовать:

var data = getApiData(rootRes);


---

11. Практический UX сценарий после этого пакета

Теперь администратор или редактор делает так:

1. открывает страницу с блоком;


2. если root нет — видит кнопки:

создать корень сайта;

создать папку блока;



3. создает нужный root;


4. открывает настройки;


5. задает:

название блока;

использовать корень сайта или свою папку;

table/grid;

ограничения;

права/видимость;



6. сохраняет;


7. блок перечитывает настройки и root;


8. интерфейс перестраивается.



Это уже близко к реальному поведению конструктора.


---

12. Что теперь у вас уже есть почти в готовом виде

После этого шага компонент уже имеет:

контекст блока;

Disk storage через Bitrix Disk;

роли сайта;

права блока;

создание корня сайта и папки блока;

список/поиск/CRUD;

модалку настроек;

сохранение настроек;

серверную валидацию sessid;

переключение root-источника.



---

13. Что логично сделать следующим шагом

Самый полезный следующий шаг теперь — интеграция в ваш editor/sitebuilder, а именно:

зарегистрировать блок disk в палитре блоков;

добавить API-создание блока type=disk;

отрисовывать его в editor.php / public.php;

дать editor-режим для блока;

привязать кнопку “Настройки” редактора к этой модалке.


Следующим сообщением я бы уже сделал сценарий полноценного подключения блока “Диск” в ваш sitebuilder:

регистрация блока,

создание блока через API,

рендер в странице,

editor/runtime режимы.