Да, добавляем блок в editor.php.

1. В state добавь accessItems

Найди:

var state = {
    site: null,
    pages: [],
    currentPageId: 0,
    blocks: [],
    currentBlockId: 0
};

Замени на:

var state = {
    site: null,
    pages: [],
    currentPageId: 0,
    blocks: [],
    currentBlockId: 0,
    accessItems: []
};


---

2. В <style> добавь стили

Вставь перед @media:

.sb-access-form {
    display: grid;
    grid-template-columns: 1fr 130px;
    gap: 10px;
    align-items: end;
}

.sb-access-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
    margin-top: 14px;
}

.sb-access-item {
    display: flex;
    justify-content: space-between;
    gap: 10px;
    align-items: center;
    padding: 10px 12px;
    border: 1px solid #e5e7eb;
    border-radius: 12px;
    background: #f9fafb;
}

.sb-access-user {
    min-width: 0;
}

.sb-access-user__name {
    font-size: 13px;
    font-weight: 700;
    color: #111827;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
}

.sb-access-user__meta {
    margin-top: 3px;
    font-size: 12px;
    color: #6b7280;
}

.sb-access-role {
    display: inline-flex;
    align-items: center;
    min-height: 24px;
    padding: 0 8px;
    border-radius: 999px;
    background: #eef2ff;
    color: #3730a3;
    font-size: 12px;
    font-weight: 700;
}

.sb-access-actions {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-shrink: 0;
}

.sb-access-message {
    margin-top: 12px;
    padding: 10px 12px;
    border-radius: 12px;
    border: 1px solid #e5e7eb;
    background: #f8fafc;
    font-size: 13px;
    color: #374151;
    line-height: 1.45;
}

.sb-access-message.is-success {
    background: #f0fdf4;
    border-color: #bbf7d0;
    color: #166534;
}

.sb-access-message.is-error {
    background: #fef2f2;
    border-color: #fecaca;
    color: #991b1b;
}


---

3. В правую колонку добавь HTML-блок прав

Лучше вставить его после блока “Группа Битрикс24 и права” и перед “Ответ API”.

<div class="sb-panel" id="siteAccessPanel" hidden>
    <h2 class="sb-panel-title">Права пользователей</h2>
    <p class="sb-editor-note">
        Выдай пользователю роль на сайте. Если у сайта есть группа Битрикс24, пользователь будет добавлен туда автоматически.
    </p>

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

    <div class="sb-editor-inspector-actions">
        <button class="sb-btn sb-btn-primary" type="button" id="grantAccessBtn">Выдать роль</button>
        <button class="sb-btn sb-btn-light" type="button" id="reloadAccessBtn">Обновить</button>
    </div>

    <div id="accessMessage" class="sb-access-message sb-hidden"></div>

    <div id="accessList" class="sb-access-list">
        <div class="sb-empty">Загрузка прав...</div>
    </div>
</div>


---

4. В JS добавь функции управления правами

Внутри <script>, рядом с другими async function, добавь:

function setAccessMessage(message, type) {
    var node = document.getElementById('accessMessage');
    if (!node) return;

    node.classList.remove('sb-hidden', 'is-success', 'is-error');

    if (type === 'success') {
        node.classList.add('is-success');
    }

    if (type === 'error') {
        node.classList.add('is-error');
    }

    node.textContent = message || '';
}

function hideAccessMessage() {
    var node = document.getElementById('accessMessage');
    if (!node) return;

    node.classList.add('sb-hidden');
    node.textContent = '';
}

function renderAccessList() {
    var panel = document.getElementById('siteAccessPanel');
    var list = document.getElementById('accessList');

    if (!panel || !list) return;

    panel.hidden = false;

    var items = Array.isArray(state.accessItems) ? state.accessItems : [];

    if (!items.length) {
        list.innerHTML = '<div class="sb-empty">Права пока не выданы</div>';
        return;
    }

    list.innerHTML = items.map(function (item) {
        var userId = Number(item.userId || 0);
        var userName = item.userName || ('Пользователь #' + userId);
        var role = item.role || '';

        return ''
            + '<div class="sb-access-item" data-access-user-id="' + userId + '">'
            + '  <div class="sb-access-user">'
            + '      <div class="sb-access-user__name">' + escapeHtml(userName) + '</div>'
            + '      <div class="sb-access-user__meta">U' + userId + ' · ' + escapeHtml(item.accessCode || '') + '</div>'
            + '  </div>'
            + '  <div class="sb-access-actions">'
            + '      <span class="sb-access-role">' + escapeHtml(role) + '</span>'
            + '      <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-access-remove-user="' + userId + '">Удалить</button>'
            + '  </div>'
            + '</div>';
    }).join('');
}

async function loadAccessList() {
    var panel = document.getElementById('siteAccessPanel');
    if (!panel) return;

    try {
        var res = await api('site.accessList', {
            siteId: siteId
        });

        state.accessItems = Array.isArray(res.items) ? res.items : [];
        renderAccessList();
    } catch (e) {
        /*
         * Если пользователь не OWNER и не админ Битрикс24,
         * API вернет ACCESS_DENIED. Это нормально — просто скрываем блок.
         */
        panel.hidden = true;
    }
}

async function grantAccessRole() {
    var userIdInput = document.getElementById('accessUserIdInput');
    var roleInput = document.getElementById('accessRoleInput');

    if (!userIdInput || !roleInput) return;

    var userId = Number(userIdInput.value || 0);
    var role = String(roleInput.value || '').trim();

    if (userId <= 0) {
        setAccessMessage('Укажи ID пользователя', 'error');
        userIdInput.focus();
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

        userIdInput.value = '';

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

async function removeAccessRole(userId) {
    userId = Number(userId || 0);

    if (userId <= 0) return;

    if (!confirm('Удалить права пользователя U' + userId + '?')) {
        return;
    }

    try {
        setAccessMessage('Удаляю права...', '');

        var res = await api('site.accessRemove', {
            siteId: siteId,
            userId: userId
        });

        state.accessItems = Array.isArray(res.items) ? res.items : [];
        renderAccessList();

        setAccessMessage('Права пользователя U' + userId + ' удалены', 'success');
    } catch (e) {
        setAccessMessage('Ошибка удаления прав: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR'), 'error');
    }
}


---

5. В init() добавь загрузку прав

Сейчас у тебя внизу:

(async function init() {
    try {
        await loadSite();
        await loadPages();
        await loadBlocks();
    } catch (e) {
        print(e);
        alert('Не удалось загрузить редактор');
    }
})();

Замени на:

(async function init() {
    try {
        await loadSite();
        await loadAccessList();
        await loadPages();
        await loadBlocks();
    } catch (e) {
        print(e);
        alert('Не удалось загрузить редактор');
    }
})();


---

6. Добавь обработчики кнопок

Внизу, рядом с остальными addEventListener, добавь:

var grantAccessBtn = document.getElementById('grantAccessBtn');
if (grantAccessBtn) {
    grantAccessBtn.addEventListener('click', grantAccessRole);
}

var reloadAccessBtn = document.getElementById('reloadAccessBtn');
if (reloadAccessBtn) {
    reloadAccessBtn.addEventListener('click', function () {
        hideAccessMessage();
        loadAccessList();
    });
}

var accessList = document.getElementById('accessList');
if (accessList) {
    accessList.addEventListener('click', function (e) {
        var btn = e.target.closest('[data-access-remove-user]');
        if (!btn) return;

        removeAccessRole(Number(btn.getAttribute('data-access-remove-user') || 0));
    });
}


---

После этого в редакторе справа появится блок «Права пользователей». Он будет доступен только владельцу сайта или администратору Битрикс24.