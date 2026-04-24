В этот editor.php лучше добавить небольшой блок справа, перед «Ответ API».

1. В <style> добавь

Вставь перед @media:

.sb-editor-group-box {
    border: 1px solid #e5e7eb;
    border-radius: 14px;
    background: #f9fafb;
    padding: 12px;
}

.sb-editor-group-row {
    display: flex;
    justify-content: space-between;
    gap: 12px;
    font-size: 13px;
    margin-bottom: 8px;
}

.sb-editor-group-row:last-child {
    margin-bottom: 0;
}

.sb-editor-group-label {
    color: #6b7280;
}

.sb-editor-group-value {
    color: #111827;
    font-weight: 700;
    text-align: right;
}

.sb-editor-sync-result {
    margin-top: 12px;
    padding: 10px 12px;
    border-radius: 12px;
    background: #f8fafc;
    border: 1px solid #e5e7eb;
    font-size: 13px;
    color: #374151;
    line-height: 1.5;
    white-space: pre-line;
}

.sb-editor-sync-result.is-success {
    background: #f0fdf4;
    border-color: #bbf7d0;
    color: #166534;
}

.sb-editor-sync-result.is-error {
    background: #fef2f2;
    border-color: #fecaca;
    color: #991b1b;
}


---

2. В правую колонку добавь HTML-блок

Найди вот этот блок:

<div class="sb-panel">
    <h2 class="sb-panel-title">Ответ API</h2>
    <div id="output" class="sb-output">Здесь будут ответы API...</div>
</div>

И перед ним вставь:

<div class="sb-panel">
    <h2 class="sb-panel-title">Группа Битрикс24 и права</h2>
    <p class="sb-editor-note">
        Участники группы Битрикс24 могут синхронизироваться с правами сайта.
    </p>

    <div class="sb-editor-group-box">
        <div class="sb-editor-group-row">
            <span class="sb-editor-group-label">ID группы</span>
            <span class="sb-editor-group-value" id="bitrixGroupIdText">—</span>
        </div>

        <div class="sb-editor-group-row">
            <span class="sb-editor-group-label">Создана</span>
            <span class="sb-editor-group-value" id="bitrixGroupCreatedAtText">—</span>
        </div>

        <div class="sb-editor-group-row">
            <span class="sb-editor-group-label">Связь</span>
            <span class="sb-editor-group-value" id="bitrixGroupStatusText">—</span>
        </div>
    </div>

    <div class="sb-editor-inspector-actions">
        <a class="sb-btn sb-btn-light" href="#" target="_blank" id="openBitrixGroupBtn">Открыть группу</a>
        <button class="sb-btn sb-btn-primary" type="button" id="syncAccessBtn">Синхронизировать права</button>
    </div>

    <div id="syncAccessResult" class="sb-editor-sync-result sb-hidden"></div>
</div>


---

3. В JS добавь функцию рендера блока группы

Внутри <script>, после функции loadSite() добавь:

function renderBitrixGroupPanel() {
    var site = state.site || {};
    var groupId = Number(site.bitrixGroupId || 0);
    var groupUrl = site.bitrixGroupUrl || (groupId > 0 ? '/workgroups/group/' + groupId + '/' : '');
    var createdAt = site.bitrixGroupCreatedAt || '';

    var groupIdText = document.getElementById('bitrixGroupIdText');
    var createdAtText = document.getElementById('bitrixGroupCreatedAtText');
    var statusText = document.getElementById('bitrixGroupStatusText');
    var openBtn = document.getElementById('openBitrixGroupBtn');
    var syncBtn = document.getElementById('syncAccessBtn');

    if (groupIdText) {
        groupIdText.textContent = groupId > 0 ? String(groupId) : 'Не создана';
    }

    if (createdAtText) {
        createdAtText.textContent = createdAt || '—';
    }

    if (statusText) {
        statusText.textContent = groupId > 0 ? 'Группа привязана' : 'Группа не привязана';
    }

    if (openBtn) {
        if (groupId > 0 && groupUrl) {
            openBtn.href = groupUrl;
            openBtn.classList.remove('sb-hidden');
        } else {
            openBtn.href = '#';
            openBtn.classList.add('sb-hidden');
        }
    }

    if (syncBtn) {
        syncBtn.disabled = groupId <= 0;
        syncBtn.title = groupId > 0 ? '' : 'Сначала нужно создать группу Битрикс24';
    }
}


---

4. Измени loadSite()

Сейчас у тебя:

async function loadSite() {
    var res = await api('site.get', {siteId: siteId});
    state.site = res.site || null;
}

Замени на:

async function loadSite() {
    var res = await api('site.get', {siteId: siteId});
    state.site = res.site || null;
    renderBitrixGroupPanel();
}


---

5. Добавь функцию синхронизации

Где-нибудь рядом с остальными async function, например после loadSite(), добавь:

async function syncAccessFromBitrixGroup() {
    var resultNode = document.getElementById('syncAccessResult');
    var btn = document.getElementById('syncAccessBtn');

    if (resultNode) {
        resultNode.classList.remove('sb-hidden', 'is-success', 'is-error');
        resultNode.textContent = 'Синхронизация...';
    }

    if (btn) {
        btn.disabled = true;
    }

    try {
        var res = await api('site.syncAccess', {
            siteId: siteId
        });

        var result = res.result || {};

        if (resultNode) {
            resultNode.classList.add('is-success');
            resultNode.textContent =
                'Права синхронизированы.\n' +
                'Создано: ' + Number(result.created || 0) + '\n' +
                'Обновлено: ' + Number(result.updated || 0) + '\n' +
                'Удалено: ' + Number(result.removed || 0) + '\n' +
                'Без изменений: ' + Number(result.kept || 0);
        }

        await loadSite();
    } catch (e) {
        if (resultNode) {
            resultNode.classList.add('is-error');
            resultNode.textContent = 'Ошибка синхронизации: ' + escapeHtml((e && (e.error || e.message)) || 'UNKNOWN_ERROR');
        }
    } finally {
        if (btn) {
            btn.disabled = false;
        }

        renderBitrixGroupPanel();
    }
}


---

6. Добавь обработчик кнопки

Внизу, рядом с другими addEventListener, после:

document.getElementById('moveBlockDownBtn').addEventListener('click', function () { moveBlock('down'); });

добавь:

var syncAccessBtn = document.getElementById('syncAccessBtn');
if (syncAccessBtn) {
    syncAccessBtn.addEventListener('click', syncAccessFromBitrixGroup);
}


---

После этого в правой колонке редактора появится блок:

Группа Битрикс24 и права
ID группы: 2
Создана: ...
Связь: Группа привязана

[Открыть группу] [Синхронизировать права]

Кнопка будет дергать:

site.syncAccess

и показывать результат синхронизации прямо в редакторе.