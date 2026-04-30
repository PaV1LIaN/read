Отлично. Тогда даю готовыми цельными кусками, чтобы ты просто вставил.


---

1) editor.css

Добавь в конец файла
/local/sitebuilder/assets/admin/editor.css

/* =========================================================
   ACCESS / ПРАВА ПОЛЬЗОВАТЕЛЕЙ
   ========================================================= */

.sb-access-selected {
    margin-top: 10px;
    padding: 12px;
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    background: #f8fafc;
}

.sb-access-selected-user {
    display: grid;
    grid-template-columns: 56px minmax(0, 1fr) auto;
    gap: 12px;
    align-items: center;
}

.sb-access-selected-avatar {
    width: 56px;
    height: 56px;
    border-radius: 50%;
    overflow: hidden;
    background: #eef2ff;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #3730a3;
    font-size: 15px;
    font-weight: 700;
    flex-shrink: 0;
}

.sb-access-selected-avatar img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
}

.sb-access-selected-body {
    min-width: 0;
}

.sb-access-selected-title {
    font-size: 14px;
    font-weight: 600;
    line-height: 1.35;
    color: #111827;
    word-break: break-word;
}

.sb-access-selected-meta {
    margin-top: 3px;
    font-size: 12px;
    line-height: 1.35;
    color: #6b7280;
    word-break: break-word;
}

.sb-access-selected-actions {
    display: flex;
    justify-content: flex-end;
    align-items: center;
    gap: 8px;
}

/* Результаты поиска */
.sb-access-search-results {
    margin-top: 8px;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    background: #fff;
    overflow: hidden;
}

.sb-access-result-item {
    width: 100%;
    display: grid;
    grid-template-columns: 32px minmax(0, 1fr);
    gap: 10px;
    align-items: center;
    padding: 8px 10px;
    border: 0;
    border-bottom: 1px solid #f1f5f9;
    background: #fff;
    text-align: left;
    cursor: pointer;
    transition: background .15s ease;
}

.sb-access-result-item:last-child {
    border-bottom: 0;
}

.sb-access-result-item:hover {
    background: #f8fafc;
}

.sb-access-result-avatar {
    width: 32px;
    height: 32px;
    border-radius: 50%;
    overflow: hidden;
    background: #eef2ff;
    display: flex;
    align-items: center;
    justify-content: center;
    color: #3730a3;
    font-size: 11px;
    font-weight: 700;
    flex-shrink: 0;
}

.sb-access-result-avatar img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
}

.sb-access-result-body {
    min-width: 0;
}

.sb-access-result-title {
    font-size: 13px;
    font-weight: 600;
    line-height: 1.25;
    color: #111827;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.sb-access-result-meta {
    margin-top: 2px;
    font-size: 11px;
    line-height: 1.25;
    color: #6b7280;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

/* Список текущих прав */
.sb-access-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 12px;
}

.sb-access-item {
    display: grid;
    grid-template-columns: minmax(0, 1fr) auto;
    gap: 10px;
    align-items: center;
    padding: 10px 12px;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    background: #fff;
}

.sb-access-item__main {
    min-width: 0;
}

.sb-access-item__name {
    font-size: 14px;
    font-weight: 600;
    color: #111827;
    line-height: 1.3;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.sb-access-item__meta {
    margin-top: 2px;
    font-size: 12px;
    color: #6b7280;
    line-height: 1.3;
    white-space: nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}

.sb-access-item__side {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
}

.sb-access-item__side .sb-btn {
    white-space: nowrap;
}

.sb-access-help {
    margin: 0 0 12px;
    font-size: 13px;
    line-height: 1.5;
    color: #6b7280;
}

/* Адаптив */
@media (max-width: 1400px) {
    .sb-access-selected-user {
        grid-template-columns: 56px minmax(0, 1fr);
    }

    .sb-access-selected-actions {
        grid-column: 1 / -1;
        justify-content: flex-start;
    }

    .sb-access-item {
        grid-template-columns: 1fr;
    }

    .sb-access-item__side {
        justify-content: flex-start;
        flex-wrap: wrap;
    }
}


---

2) editor.php — добавь поле в state

Внутри state = { ... } добавь, если его ещё нет:

selectedAccessUser: null,
userSearchResults: [],
accessItems: []

Пример:

var state = {
    site: null,
    pages: [],
    currentPageId: 0,
    blocks: [],
    currentBlockId: 0,
    selectedAccessUser: null,
    userSearchResults: [],
    accessItems: []
};


---

3) editor.php — готовые функции

Ниже идут готовые функции целиком.
Замени ими свои текущие функции, связанные с поиском/выбором пользователя и списком прав.


---

renderAccessUserSearchResults

function renderAccessUserSearchResults(users) {
    var results = document.getElementById('accessUserSearchResults');
    if (!results) return;

    state.userSearchResults = Array.isArray(users) ? users : [];

    if (!state.userSearchResults.length) {
        results.innerHTML = '';
        results.classList.add('sb-hidden');
        return;
    }

    results.innerHTML = state.userSearchResults.map(function (user) {
        var id = Number(user.id || 0);
        var title = user.title || user.name || ('Пользователь #' + id);
        var meta = [];

        if (user.login) meta.push(user.login);
        if (user.email) meta.push(user.email);

        return ''
            + '<button class="sb-access-result-item" type="button" data-select-access-user="' + id + '">'
            +      userAvatarHtml(user, 'sb-access-result-avatar')
            + '  <div class="sb-access-result-body">'
            + '      <div class="sb-access-result-title">' + escapeHtml(title) + '</div>'
            + '      <div class="sb-access-result-meta">ID: ' + id + (meta.length ? ' · ' + escapeHtml(meta.join(' · ')) : '') + '</div>'
            + '  </div>'
            + '</button>';
    }).join('');

    results.classList.remove('sb-hidden');
}


---

renderSelectedAccessUser

function renderSelectedAccessUser() {
    var selectedNode = document.getElementById('accessSelectedUser');
    if (!selectedNode) return;

    var user = state.selectedAccessUser;

    if (!user) {
        selectedNode.innerHTML = '';
        selectedNode.classList.add('sb-hidden');
        return;
    }

    var userId = Number(user.id || 0);
    var meta = [];

    if (user.login) meta.push(user.login);
    if (user.email) meta.push(user.email);

    selectedNode.innerHTML = ''
        + '<div class="sb-access-selected-user">'
        +      userAvatarHtml(user, 'sb-access-selected-avatar')
        + '  <div class="sb-access-selected-body">'
        + '      <div class="sb-access-selected-title">' + escapeHtml(user.title || user.name || ('Пользователь #' + userId)) + '</div>'
        + '      <div class="sb-access-selected-meta">ID: ' + userId + (meta.length ? ' · ' + escapeHtml(meta.join(' · ')) : '') + '</div>'
        + '  </div>'
        + '  <div class="sb-access-selected-actions">'
        + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-clear-access-user>Сбросить</button>'
        + '  </div>'
        + '</div>';

    selectedNode.classList.remove('sb-hidden');
}


---

selectAccessUser

function selectAccessUser(user) {
    state.selectedAccessUser = user || null;

    var input = document.getElementById('accessUserSearchInput');
    if (input && user) {
        input.value = user.title || user.name || '';
    }

    var results = document.getElementById('accessUserSearchResults');
    if (results) {
        results.innerHTML = '';
        results.classList.add('sb-hidden');
    }

    renderSelectedAccessUser();
}


---

clearSelectedAccessUser

function clearSelectedAccessUser() {
    state.selectedAccessUser = null;

    var input = document.getElementById('accessUserSearchInput');
    if (input) {
        input.value = '';
        input.focus();
    }

    renderSelectedAccessUser();
    renderAccessUserSearchResults([]);
}


---

renderAccessList

function renderAccessList() {
    var list = document.getElementById('accessList');
    if (!list) return;

    if (!Array.isArray(state.accessItems) || !state.accessItems.length) {
        list.innerHTML = '<div class="sb-empty">Права ещё не выданы</div>';
        return;
    }

    list.innerHTML = state.accessItems.map(function (item) {
        var userId = Number(item.userId || 0);
        var name = item.userName || item.title || ('Пользователь #' + userId);
        var role = item.role || '';

        return ''
            + '<div class="sb-access-item">'
            + '  <div class="sb-access-item__main">'
            + '      <div class="sb-access-item__name">' + escapeHtml(name) + '</div>'
            + '      <div class="sb-access-item__meta">ID: ' + userId + ' · ' + escapeHtml(item.accessCode || '') + '</div>'
            + '  </div>'
            + '  <div class="sb-access-item__side">'
            +        roleBadge(role)
            + '      <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-access-remove-user="' + userId + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }).join('');
}


---

4) editor.php — helper для аватарки

Если у тебя ещё нет функции userAvatarHtml(...), добавь её.

function userAvatarHtml(user, className) {
    className = className || '';

    var avatar = user.avatarUrl || user.avatar || user.photoUrl || '';
    var title = user.title || user.name || '';
    var initials = 'U';

    if (title) {
        var parts = String(title).trim().split(/\s+/).filter(Boolean);
        if (parts.length === 1) {
            initials = parts[0].substring(0, 1).toUpperCase();
        } else if (parts.length >= 2) {
            initials = (parts[0].substring(0, 1) + parts[1].substring(0, 1)).toUpperCase();
        }
    }

    if (avatar) {
        return '<div class="' + className + '"><img src="' + escapeHtml(avatar) + '" alt=""></div>';
    }

    return '<div class="' + className + '">' + escapeHtml(initials) + '</div>';
}


---

5) editor.php — обработчики кликов

Нужно, чтобы:

можно было выбрать пользователя из результатов поиска;

можно было нажать Сбросить;

можно было удалить пользователя из списка.


Если у тебя уже есть контейнеры, добавь/обнови обработчики.


---

Для выбора найденного пользователя

document.addEventListener('click', function (e) {
    var selectBtn = e.target.closest('[data-select-access-user]');
    if (selectBtn) {
        var userId = Number(selectBtn.getAttribute('data-select-access-user') || 0);
        var user = (state.userSearchResults || []).find(function (item) {
            return Number(item.id || 0) === userId;
        });

        if (user) {
            selectAccessUser(user);
        }
        return;
    }

    var clearBtn = e.target.closest('[data-clear-access-user]');
    if (clearBtn) {
        clearSelectedAccessUser();
        return;
    }
});


---

Для удаления пользователя из списка прав

Если у тебя уже есть свой обработчик удаления — оставь его.
Если нет, добавь такой:

document.addEventListener('click', function (e) {
    var removeBtn = e.target.closest('[data-access-remove-user]');
    if (!removeBtn) return;

    var userId = Number(removeBtn.getAttribute('data-access-remove-user') || 0);
    if (!userId || !state.currentPageId && !siteId) {
        return;
    }

    if (!confirm('Удалить права у пользователя?')) {
        return;
    }

    api('site.accessRemove', {
        siteId: siteId,
        userId: userId
    }).then(function (res) {
        var items = [];
        if (res && res.items && Array.isArray(res.items)) {
            items = res.items;
        } else if (res && res.result && Array.isArray(res.result.items)) {
            items = res.result.items;
        }

        state.accessItems = items;
        renderAccessList();
    }).catch(function (err) {
        print(err);
        alert('Не удалось удалить права');
    });
});


---

6) По HTML — что должно быть в разметке

Проверь, что в блоке прав есть такие элементы с такими id:

<input class="sb-input" type="text" id="accessUserSearchInput" placeholder="ФИО, логин, email или ID">

<div id="accessUserSearchResults" class="sb-access-search-results sb-hidden"></div>

<div id="accessSelectedUser" class="sb-access-selected sb-hidden"></div>

<div id="accessList" class="sb-access-list"></div>

Если у тебя id называются иначе — просто переименуй их в функциях.


---

7) Что получится после этих правок

Будет так:

в поиске появятся маленькие аккуратные аватарки;

карточка выбранного пользователя станет компактной и не будет ломать блок;

имя, роль и кнопка удаления в списке выданных прав больше не будут наезжать друг на друга;

длинные ФИО будут аккуратно обрезаться;

на узком экране элементы будут переноситься вниз, а не накладываться.



---

Если хочешь, следующим сообщением я могу прислать ещё и готовый цельный блок для раздела “Права пользователей” в editor.php целиком, чтобы вообще не собирать по кускам.