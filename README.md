Отлично. Тогда даю готовую интеграцию блока disk в renderer и editor.

Идем практично:

1. добавить disk в список добавления блоков в editor.php


2. дать ему нормальный preview в редакторе


3. при сохранении блока держать настройки в props


4. в публичном рендере добавить case 'disk'


5. подключить disk/styles.css и disk/script.js на странице, где есть такой блок




---

1. Что хранить в блоке disk

Для блока типа disk:

type = 'disk'

content = {}

props = настройки диска


Пример props:

{
  "title": "Документы",
  "rootMode": "site",
  "rootFolderId": null,
  "viewMode": "table",
  "allowUpload": true,
  "allowCreateFolder": true,
  "allowRename": true,
  "allowDelete": false,
  "allowDownload": true,
  "showSearch": true,
  "showBreadcrumbs": true,
  "defaultSort": "updatedAt",
  "defaultSortDirection": "desc",
  "allowedExtensions": [],
  "maxFileSize": 52428800,
  "permissionMode": "inherit_site",
  "useSiteRootFallback": true
}


---

2. Правка editor.php: добавить карточку блока disk

Найди блок с карточками добавления:

<div class="sb-editor-addbar">

И добавь туда еще одну карточку:

<button class="sb-editor-add-card" type="button" data-add-block="disk">
    <span class="sb-editor-add-card__title">Диск</span>
    <span class="sb-editor-add-card__text">Файлы, папки, загрузка и доступы в контексте сайта</span>
</button>


---

3. Правка editor.php: при создании disk блока подставлять дефолтные props

Найди функцию createBlock(type) и замени ее целиком на это:

async function createBlock(type) {
    if (!state.currentPageId) {
        alert('Сначала выберите страницу');
        return;
    }

    var content = {};
    var props = {};

    if (type === 'heading') {
        content = { text: 'Новый заголовок' };
    } else if (type === 'text') {
        content = { text: 'Новый текстовый блок' };
    } else if (type === 'button') {
        content = { label: 'Кнопка', href: '#' };
    } else if (type === 'html') {
        content = { html: '<div>Новый HTML блок</div>' };
    } else if (type === 'disk') {
        content = {};
        props = {
            title: 'Файлы',
            rootMode: 'site',
            rootFolderId: null,
            viewMode: 'table',
            allowUpload: true,
            allowCreateFolder: true,
            allowRename: true,
            allowDelete: false,
            allowDownload: true,
            showSearch: true,
            showBreadcrumbs: true,
            defaultSort: 'updatedAt',
            defaultSortDirection: 'desc',
            allowedExtensions: [],
            maxFileSize: 52428800,
            permissionMode: 'inherit_site',
            useSiteRootFallback: true
        };
    }

    await api('block.create', {
        pageId: state.currentPageId,
        type: type,
        content: JSON.stringify(content),
        props: JSON.stringify(props)
    });

    await loadBlocks();
}


---

4. Правка editor.php: дать блоку disk понятный preview

Найди функцию blockPreviewText(block) и замени ее целиком:

function blockPreviewText(block) {
    var type = String(block.type || '');
    var content = block.content || {};
    var props = block.props || {};

    if (type === 'heading') {
        return content.text || '[пустой заголовок]';
    }

    if (type === 'text') {
        return content.text || '[пустой текст]';
    }

    if (type === 'button') {
        return (content.label || 'Кнопка') + (content.href ? ' → ' + content.href : '');
    }

    if (type === 'html') {
        return (content.html || '').slice(0, 220) || '[пустой HTML]';
    }

    if (type === 'disk') {
        var title = props.title || 'Файлы';
        var mode = props.rootMode || 'site';
        var view = props.viewMode || 'table';

        return 'Компонент "Диск": ' + title + ' · rootMode=' + mode + ' · view=' + view;
    }

    try {
        return JSON.stringify(content);
    } catch (e) {
        return '[контент блока]';
    }
}


---

5. Правка editor.php: отдельная форма настроек для disk

Сейчас у тебя справа свойства блока идут только через два JSON textarea.
Для disk лучше сделать человеческую форму.

5.1. В блоке "Свойства блока" после blockTypeInput вставь HTML

Найди:

<div class="sb-field">
    <label for="blockTypeInput">Тип</label>
    <input class="sb-input" type="text" id="blockTypeInput" disabled>
</div>

И сразу после него вставь:

<div id="diskBlockForm" class="sb-hidden" style="margin-top:12px;">
    <div class="sb-field">
        <label for="diskTitleInput">Заголовок блока</label>
        <input class="sb-input" type="text" id="diskTitleInput">
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="diskRootModeInput">Режим корня</label>
        <select class="sb-select" id="diskRootModeInput">
            <option value="site">Корень сайта</option>
            <option value="block">Папка блока</option>
        </select>
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="diskViewModeInput">Вид</label>
        <select class="sb-select" id="diskViewModeInput">
            <option value="table">Таблица</option>
            <option value="grid">Плитка</option>
        </select>
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="diskPermissionModeInput">Режим прав</label>
        <select class="sb-select" id="diskPermissionModeInput">
            <option value="inherit_site">Наследовать права сайта</option>
            <option value="custom">Собственные ограничения блока</option>
        </select>
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="diskMaxFileSizeInput">Максимальный размер файла</label>
        <input class="sb-input" type="number" id="diskMaxFileSizeInput" min="0">
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="diskAllowedExtensionsInput">Разрешенные расширения</label>
        <input class="sb-input" type="text" id="diskAllowedExtensionsInput" placeholder="pdf docx xlsx png jpg">
    </div>

    <div class="sb-form-row" style="margin-top:12px;">
        <label><input type="checkbox" id="diskAllowUploadInput"> Загрузка</label>
        <label><input type="checkbox" id="diskAllowCreateFolderInput"> Создание папок</label>
    </div>

    <div class="sb-form-row" style="margin-top:12px;">
        <label><input type="checkbox" id="diskAllowRenameInput"> Переименование</label>
        <label><input type="checkbox" id="diskAllowDeleteInput"> Удаление</label>
    </div>

    <div class="sb-form-row" style="margin-top:12px;">
        <label><input type="checkbox" id="diskAllowDownloadInput"> Скачивание</label>
        <label><input type="checkbox" id="diskShowSearchInput"> Показывать поиск</label>
    </div>

    <div class="sb-form-row" style="margin-top:12px;">
        <label><input type="checkbox" id="diskShowBreadcrumbsInput"> Показывать breadcrumbs</label>
        <label><input type="checkbox" id="diskUseSiteRootFallbackInput"> Использовать корень сайта как fallback</label>
    </div>

    <div class="sb-editor-divider"></div>
</div>


---

5.2. Обнови fillBlockForm()

Найди функцию fillBlockForm() и замени ее целиком:

function fillBlockForm() {
    var block = getCurrentBlock();
    var emptyNode = document.getElementById('blockInspectorEmpty');
    var formNode = document.getElementById('blockInspector');
    var diskForm = document.getElementById('diskBlockForm');

    if (!block) {
        emptyNode.classList.remove('sb-hidden');
        formNode.classList.add('sb-hidden');
        if (diskForm) diskForm.classList.add('sb-hidden');

        document.getElementById('blockTypeInput').value = '';
        document.getElementById('blockContentInput').value = '';
        document.getElementById('blockPropsInput').value = '';
        return;
    }

    emptyNode.classList.add('sb-hidden');
    formNode.classList.remove('sb-hidden');

    document.getElementById('blockTypeInput').value = block.type || '';

    var content = block.content || {};
    var props = block.props || {};

    document.getElementById('blockContentInput').value = JSON.stringify(content, null, 2);
    document.getElementById('blockPropsInput').value = JSON.stringify(props, null, 2);

    if (block.type === 'disk') {
        if (diskForm) diskForm.classList.remove('sb-hidden');

        document.getElementById('diskTitleInput').value = props.title || 'Файлы';
        document.getElementById('diskRootModeInput').value = props.rootMode || 'site';
        document.getElementById('diskViewModeInput').value = props.viewMode || 'table';
        document.getElementById('diskPermissionModeInput').value = props.permissionMode || 'inherit_site';
        document.getElementById('diskMaxFileSizeInput').value = props.maxFileSize || 52428800;
        document.getElementById('diskAllowedExtensionsInput').value = Array.isArray(props.allowedExtensions) ? props.allowedExtensions.join(' ') : '';

        document.getElementById('diskAllowUploadInput').checked = !!props.allowUpload;
        document.getElementById('diskAllowCreateFolderInput').checked = !!props.allowCreateFolder;
        document.getElementById('diskAllowRenameInput').checked = !!props.allowRename;
        document.getElementById('diskAllowDeleteInput').checked = !!props.allowDelete;
        document.getElementById('diskAllowDownloadInput').checked = !!props.allowDownload;
        document.getElementById('diskShowSearchInput').checked = !!props.showSearch;
        document.getElementById('diskShowBreadcrumbsInput').checked = !!props.showBreadcrumbs;
        document.getElementById('diskUseSiteRootFallbackInput').checked = !!props.useSiteRootFallback;
    } else {
        if (diskForm) diskForm.classList.add('sb-hidden');
    }
}


---

5.3. Обнови saveBlock()

Найди функцию saveBlock() и замени ее целиком:

async function saveBlock() {
    var block = getCurrentBlock();
    if (!block) return;

    var content;
    var props;

    try {
        content = JSON.parse(document.getElementById('blockContentInput').value || '{}');
    } catch (e) {
        alert('Контент блока должен быть валидным JSON');
        return;
    }

    try {
        props = JSON.parse(document.getElementById('blockPropsInput').value || '{}');
    } catch (e) {
        alert('Свойства блока должны быть валидным JSON');
        return;
    }

    if (block.type === 'disk') {
        props = {
            title: document.getElementById('diskTitleInput').value.trim() || 'Файлы',
            rootMode: document.getElementById('diskRootModeInput').value,
            rootFolderId: props.rootFolderId || null,
            viewMode: document.getElementById('diskViewModeInput').value,
            permissionMode: document.getElementById('diskPermissionModeInput').value,
            maxFileSize: Number(document.getElementById('diskMaxFileSizeInput').value || 0),
            allowedExtensions: String(document.getElementById('diskAllowedExtensionsInput').value || '')
                .trim()
                .split(/\s+/)
                .filter(Boolean),

            allowUpload: document.getElementById('diskAllowUploadInput').checked,
            allowCreateFolder: document.getElementById('diskAllowCreateFolderInput').checked,
            allowRename: document.getElementById('diskAllowRenameInput').checked,
            allowDelete: document.getElementById('diskAllowDeleteInput').checked,
            allowDownload: document.getElementById('diskAllowDownloadInput').checked,
            showSearch: document.getElementById('diskShowSearchInput').checked,
            showBreadcrumbs: document.getElementById('diskShowBreadcrumbsInput').checked,
            useSiteRootFallback: document.getElementById('diskUseSiteRootFallbackInput').checked,
            defaultSort: props.defaultSort || 'updatedAt',
            defaultSortDirection: props.defaultSortDirection || 'desc'
        };

        document.getElementById('blockPropsInput').value = JSON.stringify(props, null, 2);
    }

    await api('block.update', {
        id: block.id,
        content: JSON.stringify(content),
        props: JSON.stringify(props)
    });

    await loadBlocks();
}


---

6. Добавить рендер disk в публичную страницу

Найди место, где у тебя рендерятся блоки страницы через switch ($block['type']), и добавь:

case 'disk':
    $diskProps = is_array($block['props'] ?? null) ? $block['props'] : [];
    ?>
    <div class="sb-block sb-block--disk">
        <div class="sb-disk"
             data-site-id="<?= (int)$site['id'] ?>"
             data-page-id="<?= (int)$page['id'] ?>"
             data-block-id="<?= (int)$block['id'] ?>"
             data-initial-state="<?= htmlspecialchars(json_encode([
                 'siteId' => (int)$site['id'],
                 'pageId' => (int)$page['id'],
                 'blockId' => (int)$block['id'],
                 'settings' => $diskProps,
             ], JSON_UNESCAPED_UNICODE), ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>">
        </div>
    </div>
    <?php
    break;


---

7. Подключить ассеты disk только если на странице есть блок типа disk

Перед рендером страницы проверь:

<?php
$pageHasDiskBlock = false;

foreach ($blocks as $block) {
    if (($block['type'] ?? '') === 'disk') {
        $pageHasDiskBlock = true;
        break;
    }
}
?>

И в <head> или перед </body> подключи:

<?php if ($pageHasDiskBlock): ?>
    <link rel="stylesheet" href="/local/sitebuilder/components/disk/styles.css">
    <script src="/local/sitebuilder/components/disk/script.js"></script>
<?php endif; ?>


---

8. Обновить script.js, чтобы он стартовал через bootstrap

Найди DiskComponent.prototype.init и замени целиком:

DiskComponent.prototype.init = async function () {
  this.bindStaticEvents();

  try {
    var payload = this.getBasePayload();
    payload.sessid = this.getSessid();

    var res = await this.api('bootstrap', payload);
    if (!res.ok) {
      throw new Error(res.message || res.error || 'BOOTSTRAP_ERROR');
    }

    var data = res.data || {};

    this.state.siteId = Number(data.siteId || this.state.siteId || 0);
    this.state.pageId = Number(data.pageId || this.state.pageId || 0);
    this.state.blockId = Number(data.blockId || this.state.blockId || 0);
    this.state.settings = data.settings || {};
    this.state.permissions = data.permissions || {};
    this.state.rootFolderId = data.rootFolderId || null;
    this.state.currentFolderId = data.currentFolderId || null;
    this.state.viewMode = (this.state.settings && this.state.settings.viewMode) || 'table';

    this.applyInitialViewMode();

    if (!this.state.permissions.canView) {
      this.renderState('no-access');
      return;
    }

    if (!this.state.rootFolderId) {
      this.renderState('no-root');
      return;
    }

    await this.loadFolder(this.state.rootFolderId);
  } catch (e) {
    console.error(e);
    this.renderState('error');
  }
};


---

9. Что получится после этого

После всех этих правок:

в редакторе можно создать блок disk

настройки disk живут прямо в block.props

на публичной странице блок сам знает свой siteId/pageId/blockId

при первом открытии он сам делает disk.bootstrap

root сайта или root блока создаются автоматически

права пользователя берутся из общей модели sitebuilder.access



---

10. Что логично следующим шагом

После этого уже очень правильно сделать еще 2 вещи:

1. человеческую UI-настройку блока disk в editor, без JSON textarea вообще


2. drag&drop / reorder блоков в editor, чтобы disk жил как обычный визуальный блок



Следующим сообщением я могу дать тебе полностью готовую красивую форму настроек блока disk в editor.php без JSON-полей для этого типа.