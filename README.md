Да, значит CSS где-то перебивается или картинка приходит с собственными размерами. Сделаем жёстко: и через CSS с !important, и через inline-style в editor.php.

1. В editor.css добавь в самый конец

/* =========================================================
   ЖЁСТКОЕ ИСПРАВЛЕНИЕ ПРАВ ПОЛЬЗОВАТЕЛЕЙ
   ========================================================= */

/* Поиск пользователей: компактные строки */
#accessUserSearchResults {
    display: block !important;
    max-height: 260px !important;
    overflow-y: auto !important;
    overflow-x: hidden !important;
    margin-top: 8px !important;
    border: 1px solid #e5e7eb !important;
    border-radius: 12px !important;
    background: #fff !important;
}

#accessUserSearchResults.sb-hidden {
    display: none !important;
}

#accessUserSearchResults .sb-access-result-item {
    width: 100% !important;
    min-height: 44px !important;
    display: grid !important;
    grid-template-columns: 32px minmax(0, 1fr) !important;
    gap: 10px !important;
    align-items: center !important;
    padding: 7px 10px !important;
    border: 0 !important;
    border-bottom: 1px solid #f1f5f9 !important;
    background: #fff !important;
    text-align: left !important;
    cursor: pointer !important;
    box-sizing: border-box !important;
}

#accessUserSearchResults .sb-access-result-avatar {
    width: 32px !important;
    height: 32px !important;
    min-width: 32px !important;
    max-width: 32px !important;
    min-height: 32px !important;
    max-height: 32px !important;
    border-radius: 50% !important;
    overflow: hidden !important;
    display: flex !important;
    align-items: center !important;
    justify-content: center !important;
    background: #eef2ff !important;
    color: #3730a3 !important;
    font-size: 11px !important;
    font-weight: 700 !important;
}

#accessUserSearchResults .sb-access-result-avatar img {
    width: 32px !important;
    height: 32px !important;
    max-width: 32px !important;
    max-height: 32px !important;
    min-width: 32px !important;
    min-height: 32px !important;
    object-fit: cover !important;
    display: block !important;
}

#accessUserSearchResults .sb-access-result-body {
    min-width: 0 !important;
    overflow: hidden !important;
}

#accessUserSearchResults .sb-access-result-title {
    font-size: 13px !important;
    font-weight: 700 !important;
    line-height: 1.2 !important;
    color: #111827 !important;
    white-space: nowrap !important;
    overflow: hidden !important;
    text-overflow: ellipsis !important;
}

#accessUserSearchResults .sb-access-result-meta {
    margin-top: 2px !important;
    font-size: 11px !important;
    line-height: 1.2 !important;
    color: #6b7280 !important;
    white-space: nowrap !important;
    overflow: hidden !important;
    text-overflow: ellipsis !important;
}

/* Выбранный пользователь */
#accessSelectedUser .sb-access-selected-user {
    display: grid !important;
    grid-template-columns: 42px minmax(0, 1fr) !important;
    gap: 10px !important;
    align-items: center !important;
}

#accessSelectedUser .sb-access-selected-avatar {
    width: 42px !important;
    height: 42px !important;
    min-width: 42px !important;
    max-width: 42px !important;
    min-height: 42px !important;
    max-height: 42px !important;
}

#accessSelectedUser .sb-access-selected-avatar img {
    width: 42px !important;
    height: 42px !important;
    max-width: 42px !important;
    max-height: 42px !important;
    object-fit: cover !important;
}

#accessSelectedUser .sb-access-selected-actions {
    grid-column: 1 / -1 !important;
    justify-content: flex-start !important;
}

/* Список выданных прав: всегда в 2 строки, чтобы не налезало */
#accessList .sb-access-item {
    display: grid !important;
    grid-template-columns: minmax(0, 1fr) !important;
    gap: 8px !important;
    align-items: start !important;
    padding: 10px 12px !important;
    border: 1px solid #e5e7eb !important;
    border-radius: 12px !important;
    background: #fff !important;
    box-sizing: border-box !important;
}

#accessList .sb-access-item__main {
    min-width: 0 !important;
    overflow: hidden !important;
}

#accessList .sb-access-item__name {
    font-size: 14px !important;
    font-weight: 700 !important;
    line-height: 1.25 !important;
    color: #111827 !important;
    white-space: nowrap !important;
    overflow: hidden !important;
    text-overflow: ellipsis !important;
}

#accessList .sb-access-item__meta {
    margin-top: 2px !important;
    font-size: 12px !important;
    line-height: 1.25 !important;
    color: #6b7280 !important;
    white-space: nowrap !important;
    overflow: hidden !important;
    text-overflow: ellipsis !important;
}

#accessList .sb-access-item__side {
    display: flex !important;
    align-items: center !important;
    justify-content: space-between !important;
    gap: 8px !important;
    width: 100% !important;
}

#accessList .sb-role-badge {
    flex: 0 0 auto !important;
}

#accessList .sb-access-item__side .sb-btn {
    flex: 0 0 auto !important;
    white-space: nowrap !important;
}


---

2. В editor.php замени функцию userAvatarHtml

Найди:

function userAvatarHtml(user, className) {

и замени функцию полностью:

function userAvatarHtml(user, className) {
    user = user || {};
    className = className || '';

    var avatar = user.avatarUrl || user.avatar || user.photoUrl || user.userAvatarUrl || '';
    var title = user.title || user.name || user.userName || '';
    var initials = 'U';

    if (title) {
        var parts = String(title).trim().split(/\s+/).filter(Boolean);

        if (parts.length === 1) {
            initials = parts[0].substring(0, 1).toUpperCase();
        } else if (parts.length >= 2) {
            initials = (parts[0].substring(0, 1) + parts[1].substring(0, 1)).toUpperCase();
        }
    }

    var size = '32px';

    if (className.indexOf('selected') !== -1) {
        size = '42px';
    }

    var wrapStyle = [
        'width:' + size,
        'height:' + size,
        'min-width:' + size,
        'max-width:' + size,
        'min-height:' + size,
        'max-height:' + size,
        'border-radius:50%',
        'overflow:hidden',
        'display:flex',
        'align-items:center',
        'justify-content:center',
        'background:#eef2ff',
        'color:#3730a3',
        'font-size:11px',
        'font-weight:700',
        'line-height:1'
    ].join(';');

    if (avatar) {
        return ''
            + '<div class="' + className + '" style="' + wrapStyle + '">'
            + '  <img src="' + escapeHtml(avatar) + '" alt="" style="width:' + size + ';height:' + size + ';min-width:' + size + ';max-width:' + size + ';min-height:' + size + ';max-height:' + size + ';object-fit:cover;display:block;">'
            + '</div>';
    }

    return ''
        + '<div class="' + className + '" style="' + wrapStyle + '">'
        + escapeHtml(initials)
        + '</div>';
}


---

3. В editor.php замени функцию renderAccessUserSearchResults

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
            + '<button class="sb-access-result-item" type="button" data-select-access-user="' + id + '" style="display:grid;grid-template-columns:32px minmax(0,1fr);gap:10px;align-items:center;width:100%;min-height:44px;padding:7px 10px;box-sizing:border-box;">'
            +      userAvatarHtml(user, 'sb-access-result-avatar')
            + '  <div class="sb-access-result-body" style="min-width:0;overflow:hidden;">'
            + '      <div class="sb-access-result-title" style="white-space:nowrap;overflow:hidden;text-overflow:ellipsis;">' + escapeHtml(title) + '</div>'
            + '      <div class="sb-access-result-meta" style="white-space:nowrap;overflow:hidden;text-overflow:ellipsis;">ID: ' + id + (meta.length ? ' · ' + escapeHtml(meta.join(' · ')) : '') + '</div>'
            + '  </div>'
            + '</button>';
    }).join('');

    results.classList.remove('sb-hidden');
}


---

4. В editor.php замени функцию renderAccessList

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

После этого обязательно обнови страницу с очисткой кэша:

Ctrl + F5

Или добавь версию к подключению CSS в editor.php:

<link rel="stylesheet" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/assets/admin/editor.css?v=2">

Лучше сразу поставить ?v=2, потому что браузер явно может держать старый CSS.