Да, это правильно. В редакторе обычный админ/owner не должен видеть JSON. JSON можно оставить только как технический fallback для неизвестных типов блоков.

Сделаем так:

Заголовок → обычное поле "Текст заголовка"
Текст → textarea
Кнопка → текст кнопки + ссылка + target
HTML → textarea HTML
Диск → текущая форма настроек диска
JSON → скрыт

1. В editor.php добавь стили

В <style> добавь:

.sb-block-type-form {
    display: none;
}

.sb-block-type-form.is-active {
    display: block;
}

.sb-editor-advanced-json {
    display: none;
}

.sb-editor-advanced-json.is-open {
    display: block;
}

.sb-block-form-note {
    margin: 6px 0 0;
    font-size: 12px;
    color: #6b7280;
    line-height: 1.4;
}


---

2. Замени часть инспектора блока

В editor.php найди внутри блока:

<div id="blockInspector" class="sb-hidden">

Там сейчас после поля Тип идёт diskBlockForm, потом JSON blockContentInput и blockPropsInput.

Оставь поле Тип, а после него вставь вот это:

<div id="headingBlockForm" class="sb-block-type-form" style="margin-top:12px;">
    <div class="sb-field">
        <label for="headingTextInput">Текст заголовка</label>
        <input class="sb-input" type="text" id="headingTextInput" placeholder="Введите заголовок">
    </div>
</div>

<div id="textBlockForm" class="sb-block-type-form" style="margin-top:12px;">
    <div class="sb-field">
        <label for="textTextInput">Текст блока</label>
        <textarea class="sb-textarea" id="textTextInput" placeholder="Введите текст"></textarea>
    </div>
</div>

<div id="buttonBlockForm" class="sb-block-type-form" style="margin-top:12px;">
    <div class="sb-field">
        <label for="buttonLabelInput">Текст кнопки</label>
        <input class="sb-input" type="text" id="buttonLabelInput" placeholder="Например: Подробнее">
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="buttonHrefInput">Ссылка</label>
        <input class="sb-input" type="text" id="buttonHrefInput" placeholder="https://... или /path/">
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="buttonTargetInput">Открывать</label>
        <select class="sb-select" id="buttonTargetInput">
            <option value="_self">В этом окне</option>
            <option value="_blank">В новой вкладке</option>
        </select>
    </div>
</div>

<div id="htmlBlockForm" class="sb-block-type-form" style="margin-top:12px;">
    <div class="sb-field">
        <label for="htmlInput">HTML</label>
        <textarea class="sb-textarea" id="htmlInput" placeholder="<div>HTML-код</div>"></textarea>
        <p class="sb-block-form-note">
            Используй только проверенный HTML. Скрипты лучше не вставлять.
        </p>
    </div>
</div>

<div id="unknownBlockForm" class="sb-block-type-form" style="margin-top:12px;">
    <div class="sb-empty">
        Для этого типа блока пока нет визуальной формы. Используй технический JSON ниже.
    </div>
</div>


---

3. JSON-поля спрячь в технический блок

Найди это:

<div class="sb-field" style="margin-top:12px;">
    <label for="blockContentInput">Контент (JSON)</label>
    <textarea class="sb-textarea" id="blockContentInput"></textarea>
</div>

<div class="sb-field" style="margin-top:12px;">
    <label for="blockPropsInput">Свойства (JSON)</label>
    <textarea class="sb-textarea" id="blockPropsInput"></textarea>
</div>

Замени на:

<div id="blockJsonFields" class="sb-editor-advanced-json">
    <div class="sb-field" style="margin-top:12px;">
        <label for="blockContentInput">Контент (JSON)</label>
        <textarea class="sb-textarea" id="blockContentInput"></textarea>
    </div>

    <div class="sb-field" style="margin-top:12px;">
        <label for="blockPropsInput">Свойства (JSON)</label>
        <textarea class="sb-textarea" id="blockPropsInput"></textarea>
    </div>
</div>

Если хочешь совсем убрать JSON, кнопку раскрытия не добавляем. Он будет существовать в DOM, но не отображаться.


---

4. В JS добавь функции визуальных форм

Внутри <script> добавь рядом с fillBlockForm():

function hideAllBlockTypeForms() {
    [
        'headingBlockForm',
        'textBlockForm',
        'buttonBlockForm',
        'htmlBlockForm',
        'diskBlockForm',
        'unknownBlockForm'
    ].forEach(function (id) {
        var node = document.getElementById(id);
        if (node) {
            node.classList.remove('is-active');
            node.classList.add('sb-hidden');
        }
    });
}

function showBlockTypeForm(id) {
    var node = document.getElementById(id);
    if (!node) return;

    node.classList.add('is-active');
    node.classList.remove('sb-hidden');
}

function fillVisualBlockForm(block) {
    hideAllBlockTypeForms();

    var type = String(block.type || '');
    var content = block.content || {};
    var props = block.props || {};

    if (type === 'heading') {
        showBlockTypeForm('headingBlockForm');

        var headingTextInput = document.getElementById('headingTextInput');
        if (headingTextInput) {
            headingTextInput.value = content.text || '';
        }

        return;
    }

    if (type === 'text') {
        showBlockTypeForm('textBlockForm');

        var textTextInput = document.getElementById('textTextInput');
        if (textTextInput) {
            textTextInput.value = content.text || '';
        }

        return;
    }

    if (type === 'button') {
        showBlockTypeForm('buttonBlockForm');

        var buttonLabelInput = document.getElementById('buttonLabelInput');
        var buttonHrefInput = document.getElementById('buttonHrefInput');
        var buttonTargetInput = document.getElementById('buttonTargetInput');

        if (buttonLabelInput) {
            buttonLabelInput.value = content.label || '';
        }

        if (buttonHrefInput) {
            buttonHrefInput.value = content.href || '';
        }

        if (buttonTargetInput) {
            buttonTargetInput.value = content.target || '_self';
        }

        return;
    }

    if (type === 'html') {
        showBlockTypeForm('htmlBlockForm');

        var htmlInput = document.getElementById('htmlInput');
        if (htmlInput) {
            htmlInput.value = content.html || '';
        }

        return;
    }

    if (type === 'disk') {
        showBlockTypeForm('diskBlockForm');
        return;
    }

    showBlockTypeForm('unknownBlockForm');

    var jsonFields = document.getElementById('blockJsonFields');
    if (jsonFields) {
        jsonFields.classList.add('is-open');
    }
}

function collectVisualBlockData(block) {
    var type = String(block.type || '');
    var content = {};
    var props = block.props || {};

    if (type === 'heading') {
        content = {
            text: (document.getElementById('headingTextInput')?.value || '').trim()
        };

        return {
            content: content,
            props: props
        };
    }

    if (type === 'text') {
        content = {
            text: document.getElementById('textTextInput')?.value || ''
        };

        return {
            content: content,
            props: props
        };
    }

    if (type === 'button') {
        content = {
            label: (document.getElementById('buttonLabelInput')?.value || '').trim() || 'Кнопка',
            href: (document.getElementById('buttonHrefInput')?.value || '').trim() || '#',
            target: document.getElementById('buttonTargetInput')?.value || '_self'
        };

        return {
            content: content,
            props: props
        };
    }

    if (type === 'html') {
        content = {
            html: document.getElementById('htmlInput')?.value || ''
        };

        return {
            content: content,
            props: props
        };
    }

    if (type === 'disk') {
        return {
            content: block.content || {},
            props: collectDiskBlockProps(block)
        };
    }

    try {
        content = JSON.parse(document.getElementById('blockContentInput').value || '{}');
    } catch (e) {
        alert('Контент блока должен быть валидным JSON');
        return null;
    }

    try {
        props = JSON.parse(document.getElementById('blockPropsInput').value || '{}');
    } catch (e) {
        alert('Свойства блока должны быть валидным JSON');
        return null;
    }

    return {
        content: content,
        props: props
    };
}

function collectDiskBlockProps(block) {
    var oldProps = block.props || {};

    return {
        title: document.getElementById('diskTitleInput').value.trim() || 'Файлы',
        rootMode: document.getElementById('diskRootModeInput').value,
        rootFolderId: oldProps.rootFolderId || null,
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
        defaultSort: oldProps.defaultSort || 'updatedAt',
        defaultSortDirection: oldProps.defaultSortDirection || 'desc'
    };
}


---

5. В fillBlockForm() добавь вызов визуальной формы

В функции fillBlockForm() после заполнения:

document.getElementById('blockContentInput').value = JSON.stringify(content, null, 2);
document.getElementById('blockPropsInput').value = JSON.stringify(props, null, 2);

добавь:

var jsonFields = document.getElementById('blockJsonFields');
if (jsonFields) {
    jsonFields.classList.remove('is-open');
}

fillVisualBlockForm(block);

А старую логику:

if (block.type === 'disk') {
    ...
} else {
    if (diskForm) diskForm.classList.add('sb-hidden');
}

оставь, но лучше удалить обёртку if/else, чтобы заполнение disk-полей было отдельно. Если хочешь проще — оставь старый if (block.type === 'disk'), но в конце внутри него добавь:

showBlockTypeForm('diskBlockForm');


---

6. Замени saveBlock()

Полностью замени функцию saveBlock() на эту:

async function saveBlock() {
    var block = getCurrentBlock();
    if (!block) return;

    var collected = collectVisualBlockData(block);

    if (!collected) {
        return;
    }

    await api('block.update', {
        id: block.id,
        content: JSON.stringify(collected.content),
        props: JSON.stringify(collected.props)
    });

    await loadBlocks();
}


---

После этого при выборе блока справа будет не JSON, а нормальная форма:

Заголовок:
[Текст заголовка]

Текст:
[Большое поле текста]

Кнопка:
[Текст кнопки]
[Ссылка]
[Открывать]

Диск:
[Настройки диска]

JSON останется скрытым и будет использоваться только если попадётся неизвестный тип блока.