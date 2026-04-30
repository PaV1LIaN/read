Да, оставляем диск/папки как было. Сейчас лучше сделать права пользователей аккуратнее:

1. правую колонку сделать шире;


2. список пользователей сделать карточками;


3. добавить аватар пользователя, если в Битрикс24 заполнено фото;


4. если фото нет — показывать кружок с инициалами.




---

1. В editor.css добавь в конец

Файл:

/local/sitebuilder/assets/admin/editor.css

Добавь:

/* Более широкая правая колонка редактора */
.sb-editor-shell {
    grid-template-columns: 320px minmax(0, 1fr) 440px;
}

@media (max-width: 1440px) {
    .sb-editor-shell {
        grid-template-columns: 290px minmax(0, 1fr) 400px;
    }
}

/* Права пользователей: аккуратные карточки */
.sb-access-form {
    grid-template-columns: 1fr !important;
}

.sb-access-list {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.sb-access-item {
    display: grid;
    grid-template-columns: 42px minmax(0, 1fr) auto;
    gap: 10px;
    align-items: center;
    padding: 10px;
    border: 1px solid #eef2f7;
    border-radius: 14px;
    background: #fff;
    overflow: hidden;
}

.sb-access-item__avatar,
.sb-access-result-avatar,
.sb-access-selected-avatar {
    width: 38px;
    height: 38px;
    border-radius: 999px;
    overflow: hidden;
    flex: 0 0 auto;
    background: #eef2ff;
    color: #3730a3;
    display: inline-flex;
    align-items: center;
    justify-content: center;
    font-size: 13px;
    font-weight: 800;
    text-transform: uppercase;
}

.sb-access-item__avatar img,
.sb-access-result-avatar img,
.sb-access-selected-avatar img {
    width: 100%;
    height: 100%;
    object-fit: cover;
    display: block;
}

.sb-access-item__main {
    min-width: 0;
}

.sb-access-item__name {
    font-size: 14px;
    font-weight: 700;
    color: #111827;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-item__meta {
    margin-top: 3px;
    font-size: 12px;
    color: #6b7280;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-item__side {
    display: flex;
    align-items: center;
    gap: 8px;
}

.sb-access-item__side .sb-btn {
    white-space: nowrap;
}

/* Выпадающий поиск пользователей */
.sb-access-result-item {
    display: flex;
    align-items: center;
    gap: 10px;
}

.sb-access-result-body {
    min-width: 0;
}

.sb-access-result-title {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-result-meta {
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

/* Выбранный пользователь */
.sb-access-selected {
    padding: 10px;
}

.sb-access-selected-user {
    display: flex;
    align-items: center;
    gap: 10px;
}

.sb-access-selected-body {
    min-width: 0;
}

.sb-access-selected-title {
    font-size: 13px;
    font-weight: 800;
    color: #166534;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-selected-meta {
    margin-top: 3px;
    font-size: 12px;
    color: #166534;
    opacity: .85;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-selected-actions {
    margin-left: auto;
    flex: 0 0 auto;
}

@media (max-width: 1500px) {
    .sb-access-item {
        grid-template-columns: 38px minmax(0, 1fr);
    }

    .sb-access-item__side {
        grid-column: 1 / -1;
        justify-content: flex-start;
        padding-left: 48px;
    }
}


---

2. В editor.php добавь JS-функции для аватара

Внутри <script> рядом с escapeHtml() добавь:

function getInitials(value) {
    value = String(value || '').trim();

    if (!value) {
        return '?';
    }

    var parts = value.split(/\s+/).filter(Boolean);

    if (parts.length >= 2) {
        return (parts[0].charAt(0) + parts[1].charAt(0)).toUpperCase();
    }

    return value.substring(0, 2).toUpperCase();
}

function userAvatarHtml(user, className) {
    user = user || {};

    var name = user.title || user.userName || user.name || '';
    var avatarUrl = user.avatarUrl || user.userAvatarUrl || user.photoUrl || '';

    className = className || 'sb-access-item__avatar';

    if (avatarUrl) {
        return ''
            + '<div class="' + className + '">'
            + '  <img src="' + escapeHtml(avatarUrl) + '" alt="">'
            + '</div>';
    }

    return ''
        + '<div class="' + className + '">'
        + escapeHtml(getInitials(name))
        + '</div>';
}


---

3. В editor.php замени renderAccessUserSearchResults()

Найди функцию:

function renderAccessUserSearchResults(users) {

и внутри неё замени формирование results.innerHTML = state.userSearchResults.map... на это:

results.innerHTML = state.userSearchResults.map(function (user) {
    var id = Number(user.id || 0);
    var title = user.title || ('Пользователь #' + id);
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


---

4. В editor.php замени кусок в selectAccessUser()

В функции selectAccessUser() найди:

selectedNode.innerHTML = ''
    + '<strong>' + escapeHtml(user.title || ('Пользователь #' + userId)) + '</strong>'
    + '<br>ID: ' + userId
    + (meta.length ? ' · ' + escapeHtml(meta.join(' · ')) : '')
    + ' <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-clear-access-user>Сбросить</button>';

Замени на:

selectedNode.innerHTML = ''
    + '<div class="sb-access-selected-user">'
    +      userAvatarHtml(user, 'sb-access-selected-avatar')
    + '  <div class="sb-access-selected-body">'
    + '      <div class="sb-access-selected-title">' + escapeHtml(user.title || ('Пользователь #' + userId)) + '</div>'
    + '      <div class="sb-access-selected-meta">ID: ' + userId + (meta.length ? ' · ' + escapeHtml(meta.join(' · ')) : '') + '</div>'
    + '  </div>'
    + '  <div class="sb-access-selected-actions">'
    + '      <button class="sb-btn sb-btn-light sb-btn-small" type="button" data-clear-access-user>Сбросить</button>'
    + '  </div>'
    + '</div>';


---

5. В editor.php замени renderAccessList()

Найди функцию:

function renderAccessList() {

и внутри неё замени list.innerHTML = state.accessItems.map... на это:

list.innerHTML = state.accessItems.map(function (item) {
    var userId = Number(item.userId || 0);
    var name = item.userName || item.title || ('Пользователь #' + userId);
    var role = item.role || '';

    var avatarUser = {
        userName: name,
        title: name,
        avatarUrl: item.avatarUrl || item.userAvatarUrl || item.photoUrl || ''
    };

    return ''
        + '<div class="sb-access-item">'
        +      userAvatarHtml(avatarUser, 'sb-access-item__avatar')
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

После этого список уже будет выглядеть намного лучше даже без фото — будут инициалы.


---

6. Чтобы реальные фото приходили в поиске, поправь user.php

Файл:

/local/sitebuilder/api/handlers/user.php

В функцию sb_user_search_normalize() добавь получение фото.

Найди:

$email = (string)($row['EMAIL'] ?? '');

ниже добавь:

$photoId = (int)($row['PERSONAL_PHOTO'] ?? 0);
$avatarUrl = '';

if ($photoId > 0 && class_exists('CFile')) {
    $avatarUrl = (string)CFile::GetPath($photoId);
}

И в return [ добавь:

'avatarUrl' => $avatarUrl,
'photoUrl' => $avatarUrl,

Должно быть примерно так:

return [
    'id' => $id,
    'name' => $fio,
    'login' => $login,
    'email' => $email,
    'title' => $title,
    'avatarUrl' => $avatarUrl,
    'photoUrl' => $avatarUrl,
    'active' => (string)($row['ACTIVE'] ?? ''),
];

Теперь во всех местах, где в CUser::GetList есть FIELDS, добавь туда:

'PERSONAL_PHOTO',

Например было:

'FIELDS' => [
    'ID',
    'LOGIN',
    'EMAIL',
    'NAME',
    'LAST_NAME',
    'SECOND_NAME',
    'ACTIVE',
],

Сделай:

'FIELDS' => [
    'ID',
    'LOGIN',
    'EMAIL',
    'NAME',
    'LAST_NAME',
    'SECOND_NAME',
    'PERSONAL_PHOTO',
    'ACTIVE',
],


---

Фото в уже выданных правах тоже можно добавить, но для этого нужно поправить SiteAccessManagementService.php, который формирует site.accessList. Сейчас после правок хотя бы поиск будет с фото, а в списке будет аккуратный fallback с инициалами.