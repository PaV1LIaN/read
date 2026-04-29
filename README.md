Да, дальше делаем **поиск пользователя в правах** и одновременно убираем админский шум из `editor.php` для обычного `EDITOR`.

## 1. В `editor.php` скрой админские элементы по умолчанию

Кнопку удаления сайта замени:

```html
<button class="sb-btn sb-btn-danger sb-btn-small" type="button" id="deleteSiteBtn">Удалить сайт</button>
```

на:

```html
<button class="sb-btn sb-btn-danger sb-btn-small sb-hidden" type="button" id="deleteSiteBtn">Удалить сайт</button>
```

У панели группы замени открывающий div:

```html
<div class="sb-panel">
    <h2 class="sb-panel-title">Группа Битрикс24 и права</h2>
```

на:

```html
<div class="sb-panel" id="siteGroupPanel" hidden>
    <h2 class="sb-panel-title">Группа Битрикс24 и права</h2>
```

Панель API замени:

```html
<div class="sb-panel">
    <h2 class="sb-panel-title">Ответ API</h2>
```

на:

```html
<div class="sb-panel" id="apiOutputPanel" hidden>
    <h2 class="sb-panel-title">Ответ API</h2>
```

---

## 2. В CSS добавь стили поиска пользователя

В `<style>` добавь:

```css
.sb-access-search-wrap {
    position: relative;
}

.sb-access-results {
    display: none;
    position: absolute;
    z-index: 1000;
    left: 0;
    right: 0;
    top: calc(100% + 6px);
    max-height: 260px;
    overflow: auto;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    background: #fff;
    box-shadow: 0 16px 40px rgba(15, 23, 42, 0.16);
}

.sb-access-results.is-open {
    display: block;
}

.sb-access-result-item {
    width: 100%;
    display: block;
    text-align: left;
    padding: 10px 12px;
    border: 0;
    background: #fff;
    cursor: pointer;
    border-bottom: 1px solid #f3f4f6;
}

.sb-access-result-item:hover {
    background: #f8fafc;
}

.sb-access-result-title {
    font-size: 13px;
    font-weight: 700;
    color: #111827;
}

.sb-access-result-meta {
    margin-top: 3px;
    font-size: 12px;
    color: #6b7280;
}

.sb-access-selected {
    margin-top: 8px;
    padding: 9px 10px;
    border-radius: 12px;
    border: 1px solid #bbf7d0;
    background: #f0fdf4;
    color: #166534;
    font-size: 13px;
    line-height: 1.4;
}

.sb-access-selected button {
    margin-left: 8px;
}
```

---

## 3. Замени форму прав пользователей

В блоке `siteAccessPanel` найди:

```html
<div class="sb-access-form">
    <div class="sb-field">
        <label for="accessUserIdInput">ID пользователя</label>
        <input class="sb-input" type="number" id="accessUserIdInput" min="1" placeholder="Например: 99">
    </div>

    <div class="sb-field">
        <label for="accessRoleInput">Роль</label>
        <select class="sb-select" id="accessRoleInput">
            <option value="VIEWER">VIEWER</option>
            <option value="EDITOR">EDITOR</option>
            <option value="ADMIN">ADMIN</option>
            <option value="OWNER">OWNER</option>
        </select>
    </div>
</div>
```

Замени на:

```html
<div class="sb-access-form">
    <div class="sb-field sb-access-search-wrap">
        <label for="accessUserSearchInput">Пользователь</label>
        <input class="sb-input" type="text" id="accessUserSearchInput" autocomplete="off" placeholder="ФИО, логин, email или ID">
        <input type="hidden" id="accessUserIdInput" value="">

        <div id="accessUserSearchResults" class="sb-access-results"></div>
        <div id="accessSelectedUser" class="sb-access-selected sb-hidden"></div>
    </div>

    <div class="sb-field">
        <label for="accessRoleInput">Роль</label>
        <select class="sb-select" id="accessRoleInput">
            <option value="VIEWER">VIEWER</option>
            <option value="EDITOR">EDITOR</option>
            <option value="ADMIN">ADMIN</option>
            <option value="OWNER">OWNER</option>
        </select>
    </div>
</div>
```

---

## 4. В `state` добавь поля поиска

Было:

```js
var state = {
    site: null,
    pages: [],
    currentPageId: 0,
    blocks: [],
    currentBlockId: 0,
    accessItems: []
};
```

Сделай:

```js
var state = {
    site: null,
    pages: [],
    currentPageId: 0,
    blocks: [],
    currentBlockId: 0,
    accessItems: [],
    userSearchResults: [],
    selectedAccessUser: null,
    userSearchTimer: null
};
```

---

## 5. Добавь функцию видимости админских панелей

В JS добавь рядом с функциями прав:

```js
function setManagementPanelsVisible(canManage) {
    var groupPanel = document.getElementById('siteGroupPanel');
    var accessPanel = document.getElementById('siteAccessPanel');
    var apiPanel = document.getElementById('apiOutputPanel');
    var deleteSiteBtn = document.getElementById('deleteSiteBtn');

    if (groupPanel) {
        groupPanel.hidden = !canManage;
    }

    if (accessPanel) {
        accessPanel.hidden = !canManage;
    }

    if (apiPanel) {
        apiPanel.hidden = !canManage;
    }

    if (deleteSiteBtn) {
        deleteSiteBtn.classList.toggle('sb-hidden', !canManage);
    }
}
```

---

## 6. Замени `loadAccessList()`

Найди функцию `loadAccessList()` и замени полностью:

```js
async function loadAccessList() {
    var panel = document.getElementById('siteAccessPanel');
    if (!panel) return;

    try {
        var res = await api('site.accessList', {
            siteId: siteId
        });

        state.accessItems = Array.isArray(res.items) ? res.items : [];

        setManagementPanelsVisible(true);
        renderBitrixGroupPanel();
        renderAccessList();
    } catch (e) {
        state.accessItems = [];
        setManagementPanelsVisible(false);
    }
}
```

Теперь обычный `EDITOR` не увидит:

```text
Группа Битрикс24
Права пользователей
Ответ API
Удалить сайт
```

---

## 7. Добавь функции поиска пользователя

В JS добавь:

```js
function clearAccessUserSearchResults() {
    var results = document.getElementById('accessUserSearchResults');
    if (!results) return;

    results.classList.remove('is-open');
    results.innerHTML = '';
}

function renderAccessUserSearchResults(users) {
    var results = document.getElementById('accessUserSearchResults');
    if (!results) return;

    state.userSearchResults = Array.isArray(users) ? users : [];

    if (!state.userSearchResults.length) {
        results.innerHTML = ''
            + '<div class="sb-access-result-item">'
            + '  <div class="sb-access-result-title">Ничего не найдено</div>'
            + '  <div class="sb-access-result-meta">Попробуй другой запрос</div>'
            + '</div>';
        results.classList.add('is-open');
        return;
    }

    results.innerHTML = state.userSearchResults.map(function (user) {
        var id = Number(user.id || 0);
        var title = user.title || ('Пользователь #' + id);
        var meta = [];

        if (user.login) meta.push(user.login);
        if (user.email) meta.push(user.email);

        return ''
            + '<button class="sb-access-result-item" type="button" data-select-access-user="' + id + '">'
            + '  <div class="sb-access-result-title">' + escapeHtml(title) + '</div>'
            + '  <div class="sb-access-result-meta">ID: ' + id + (meta.length ? ' · ' + escapeHtml(meta.join(' · ')) : '') + '</div>'
            + '</button>';
    }).join('');

    results.classList.add('is-open');
}

async function searchAccessUsers() {
    var input = document.getElementById('accessUserSearchInput');
    if (!input) return;

    var query = String(input.value || '').trim();

    state.selectedAccessUser = null;

    var hidden = document.getElementById('accessUserIdInput');
    if (hidden) {
        hidden.value = '';
    }

    var selectedNode = document.getElementById('accessSelectedUser');
    if (selectedNode) {
        selectedNode.classList.add('sb-hidden');
        selectedNode.innerHTML = '';
    }

    if (query === '') {
        clearAccessUserSearchResults();
        return;
    }

    if (!/^\d+$/.test(query) && query.length < 2) {
        clearAccessUserSearchResults();
        return;
    }

    try {
        var res = await api('user.search', {
            siteId: siteId,
            query: query,
            limit: 10
        });

        renderAccessUserSearchResults(Array.isArray(res.users) ? res.users : []);
    } catch (e) {
        renderAccessUserSearchResults([]);
    }
}

function selectAccessUser(userId) {
    userId = Number(userId || 0);

    if (userId <= 0) return;

    var user = state.userSearchResults.find(function (u) {
        return Number(u.id || 0) === userId;
    });

    if (!user) {
        return;
    }

    state.selectedAccessUser = user;

    var hidden = document.getElementById('accessUserIdInput');
    if (hidden) {
        hidden.value = String(userId);
    }

    var input = document.getElementById('accessUserSearchInput');
    if (input) {
        input.value = user.title || ('Пользователь #' + userId);
    }

    var selectedNode = document.getElementById('accessSelectedUser');
    if (selectedNode) {
        var meta = [];

        if (user.login) meta.push(user.login);
        if (user.email) meta.push(user.email);

        selectedNode.innerHTML = ''
            + '<strong>' + escapeHtml(user.title || ('Пользователь #' + userId)) + '</strong>'
            + '<br>ID: ' + userId
            + (meta.length ? ' · ' + escapeHtml(meta.join(' · ')) : '')
            + ' <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-clear-access-user>Сбросить</button>';

        selectedNode.classList.remove('sb-hidden');
    }

    clearAccessUserSearchResults();
}

function clearSelectedAccessUser() {
    state.selectedAccessUser = null;

    var hidden = document.getElementById('accessUserIdInput');
    if (hidden) {
        hidden.value = '';
    }

    var input = document.getElementById('accessUserSearchInput');
    if (input) {
        input.value = '';
        input.focus();
    }

    var selectedNode = document.getElementById('accessSelectedUser');
    if (selectedNode) {
        selectedNode.classList.add('sb-hidden');
        selectedNode.innerHTML = '';
    }

    clearAccessUserSearchResults();
}
```

---

## 8. Замени `grantAccessRole()`

Найди функцию `grantAccessRole()` и замени полностью:

```js
async function grantAccessRole() {
    var userIdInput = document.getElementById('accessUserIdInput');
    var roleInput = document.getElementById('accessRoleInput');

    if (!userIdInput || !roleInput) return;

    var userId = Number(userIdInput.value || 0);
    var role = String(roleInput.value || '').trim();

    if (userId <= 0) {
        setAccessMessage('Сначала найди и выбери пользователя из списка', 'error');

        var searchInput = document.getElementById('accessUserSearchInput');
        if (searchInput) {
            searchInput.focus();
        }

        return;
    }

    if (!role) {
        setAccessMessage('Выбери роль', 'error');
        return;
    }

    try {
        setAccessMessage('Сохраняю права...', '');

        var res = await api('site.accessSet', {
            siteId: siteId,
            userId: userId,
            role: role
        });

        state.accessItems = Array.isArray(res.items) ? res.items : [];

        clearSelectedAccessUser();
        renderAccessList();

        var groupSync = res.result && res.result.groupSync ? res.result.groupSync : null;
        var syncText = '';

        if (groupSync) {
            if (groupSync.ok) {
                syncText = '\nПользователь также синхронизирован с группой Битрикс24.';
            } else if (groupSync.error) {
                syncText = '\nНо с группой Битрикс24 не синхронизировался: ' + groupSync.error;
            } else if (groupSync.message) {
                syncText = '\nГруппа Битрикс24: ' + groupSync.message;
            }
        }

        setAccessMessage('Роль выдана: U' + userId + ' → ' + role + syncText, 'success');
    } catch (e) {
        setAccessMessage('Ошибка выдачи роли: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    }
}
```

---

## 9. Добавь обработчики поиска

Внизу, рядом с обработчиками кнопок, добавь:

```js
var accessUserSearchInput = document.getElementById('accessUserSearchInput');
if (accessUserSearchInput) {
    accessUserSearchInput.addEventListener('input', function () {
        clearTimeout(state.userSearchTimer);

        state.userSearchTimer = setTimeout(function () {
            searchAccessUsers();
        }, 300);
    });

    accessUserSearchInput.addEventListener('keydown', function (e) {
        if (e.key === 'Enter') {
            e.preventDefault();

            if (state.userSearchResults.length) {
                selectAccessUser(Number(state.userSearchResults[0].id || 0));
            }
        }
    });
}

var accessUserSearchResults = document.getElementById('accessUserSearchResults');
if (accessUserSearchResults) {
    accessUserSearchResults.addEventListener('click', function (e) {
        var btn = e.target.closest('[data-select-access-user]');
        if (!btn) return;

        selectAccessUser(Number(btn.getAttribute('data-select-access-user') || 0));
    });
}

var accessSelectedUser = document.getElementById('accessSelectedUser');
if (accessSelectedUser) {
    accessSelectedUser.addEventListener('click', function (e) {
        var btn = e.target.closest('[data-clear-access-user]');
        if (!btn) return;

        clearSelectedAccessUser();
    });
}

document.addEventListener('mousedown', function (e) {
    var wrap = e.target.closest('.sb-access-search-wrap');
    if (!wrap) {
        clearAccessUserSearchResults();
    }
});
```

---

После этих правок в `editor.php` блок прав станет нормальным:

```text
Пользователь: [иванов]
↓ выпадающий список
Иванов Иван — ivanov@example.ru
[выбор]

Роль: EDITOR
[Выдать роль]
```

А обычный редактор больше не будет видеть административные панели.
