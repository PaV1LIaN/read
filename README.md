Проблема, скорее всего, не в API.

На сервере вложенность уже поддерживается:

page.create принимает parentId и сохраняет его в страницу. 

page.updateMeta тоже умеет менять parentId. 

отдельно есть page.setParent. 


То есть если вложенность “не работает”, то чаще всего ломается в editor.php:

1. либо при создании страницы не отправляется parentId,


2. либо при редактировании страницы нельзя поменять родителя,


3. либо дерево слева рисуется плоско.



Ниже даю нормальный фикс для editor.php.


---

Что нужно исправить в editor.php

1. Добавить выбор родителя в “Свойства страницы”

Найди блок:

<div class="sb-field" style="margin-top:12px;">
    <label for="pageStatusInput">Статус</label>
    <select class="sb-select" id="pageStatusInput">
        <option value="draft">draft</option>
        <option value="published">published</option>
    </select>
</div>

И сразу после него вставь:

<div class="sb-field" style="margin-top:12px;">
    <label for="pageParentInput">Родительская страница</label>
    <select class="sb-select" id="pageParentInput">
        <option value="0">Без родителя</option>
    </select>
</div>


---

2. Добавить JS-функцию заполнения списка родителей

В editor.php внутри <script> вставь функцию:

function fillPageParentEditorOptions() {
    var select = document.getElementById('pageParentInput');
    if (!select) return;

    var currentPageId = Number(state.currentPageId || 0);
    var currentValue = String(select.value || '0');

    var html = '<option value="0">Без родителя</option>';

    state.pages.forEach(function (page) {
        var id = Number(page.id || 0);

        if (id === currentPageId) {
            return;
        }

        html += '<option value="' + id + '">' + escapeHtml(page.title || ('Страница #' + id)) + '</option>';
    });

    select.innerHTML = html;

    if (currentValue && select.querySelector('option[value="' + currentValue + '"]')) {
        select.value = currentValue;
    }
}


---

3. Обновить fillPageForm()

Найди функцию:

function fillPageForm() {
    var page = getCurrentPage();

    document.getElementById('pageTitleInput').value = page ? (page.title || '') : '';
    document.getElementById('pageSlugInput').value = page ? (page.slug || '') : '';
    document.getElementById('pageStatusInput').value = page ? (page.status || 'draft') : 'draft';
}

И замени целиком на это:

function fillPageForm() {
    var page = getCurrentPage();

    fillPageParentEditorOptions();

    document.getElementById('pageTitleInput').value = page ? (page.title || '') : '';
    document.getElementById('pageSlugInput').value = page ? (page.slug || '') : '';
    document.getElementById('pageStatusInput').value = page ? (page.status || 'draft') : 'draft';

    var parentSelect = document.getElementById('pageParentInput');
    if (parentSelect) {
        parentSelect.value = page ? String(page.parentId || 0) : '0';
    }
}


---

4. Обновить savePage()

Сейчас у тебя при сохранении страницы, скорее всего, меняются только title/slug/status.
Нужно еще отправлять parentId.

Найди функцию savePage() и замени ее целиком на это:

async function savePage() {
    if (!state.currentPageId) return;

    var parentId = Number(document.getElementById('pageParentInput').value || 0);

    await api('page.updateMeta', {
        id: state.currentPageId,
        title: document.getElementById('pageTitleInput').value.trim(),
        slug: document.getElementById('pageSlugInput').value.trim(),
        parentId: parentId
    });

    await api('page.setStatus', {
        id: state.currentPageId,
        status: document.getElementById('pageStatusInput').value
    });

    await loadPages();
    await loadBlocks();
}

Это как раз использует уже существующую серверную поддержку parentId в page.updateMeta. 


---

5. Убедиться, что при создании страницы реально уходит parentId

На сервере page.create уже принимает parentId. 

Поэтому функция создания на фронте должна быть такой:

async function createPage() {
    var title = (document.getElementById('newPageTitle').value || '').trim();
    var slug = (document.getElementById('newPageSlug').value || '').trim();
    var parentId = Number(document.getElementById('newPageParentId').value || 0);

    if (!title) {
        alert('Введите название страницы');
        return;
    }

    await api('page.create', {
        siteId: siteId,
        title: title,
        slug: slug,
        parentId: parentId
    });

    document.getElementById('newPageTitle').value = '';
    document.getElementById('newPageSlug').value = '';
    document.getElementById('newPageParentId').value = '0';

    await loadPages();
    await loadBlocks();
}

Если у тебя сейчас там идет отдельный page.setParent после создания — это хуже и не нужно, потому что backend уже умеет принять parentId сразу при создании. 


---

6. Проверить, что дерево страниц рисуется именно по parentId

Слева у тебя должен быть tree-builder примерно такого вида:

function buildPageTree(pages, parentId, depth, result) {
    result = result || [];
    depth = depth || 0;

    var branch = pages
        .filter(function (page) {
            return Number(page.parentId || 0) === Number(parentId || 0);
        })
        .sort(function (a, b) {
            var sortCmp = Number(a.sort || 0) - Number(b.sort || 0);
            if (sortCmp !== 0) return sortCmp;
            return Number(a.id || 0) - Number(b.id || 0);
        });

    branch.forEach(function (page) {
        result.push({
            page: page,
            depth: depth
        });
        buildPageTree(pages, Number(page.id || 0), depth + 1, result);
    });

    return result;
}

И рендер должен использовать depth, например:

style="margin-left:' + (depth * 18) + 'px;"

Если у тебя там обычный state.pages.map(...) без tree-builder, тогда вложенность в данных есть, но визуально не отображается.


---

Что проверить после правки

Сразу проверь 3 сценария:

1. Создание дочерней страницы

выбери родителя в newPageParentId

создай страницу

она должна уйти под выбранного родителя


2. Перенос существующей страницы под другую

выбери страницу

справа поменяй “Родительская страница”

нажми “Сохранить страницу”

дерево должно перестроиться


3. Снятие вложенности

выбери вложенную страницу

поставь “Без родителя”

сохрани

страница должна стать корневой



---

Если после этого все равно не заработает

Тогда уже надо смотреть не editor, а фактические данные, которые приходят после page.list.

Открой page.list в ответе API и проверь, есть ли у дочерней страницы:

{
  "parentId": 123
}

Потому что backend поддержку parentId уже содержит.

Если хочешь, я могу следующим сообщением прислать тебе полностью готовый фрагмент editor.php для блока “Свойства страницы” и функций createPage / savePage / fillPageForm одним куском, чтобы ты просто заменил его без ручной сборки.