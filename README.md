Да, возвращаемся к этапу «Создать группу Битрикс24 для старых сайтов».

Сейчас новые сайты создаются с группой автоматически, а старые сайты могут быть с:

bitrixGroupId = 0

Сделаем кнопку в редакторе:

Создать группу Битрикс24

Она будет видна только если у сайта группы ещё нет.


---

1. В api/index.php добавь action

Файл:

/local/sitebuilder/api/index.php

В список site actions добавь:

'site.ensureGroup',

Пример:

$siteActions = [
    'site.list',
    'site.get',
    'site.create',
    'site.update',
    'site.delete',
    'site.setHome',
    'site.syncAccess',
    'site.ensureGroup',
];


---

2. В site.php добавь action site.ensureGroup

Файл:

/local/sitebuilder/api/handlers/site.php

Вставь этот блок перед финальным:

sb_json_error('NOT_MOVED_YET', 501, [

Код:

if ($action === 'site.ensureGroup') {
    $siteId = (int)($_POST['siteId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    sb_require_owner($siteId);

    $site = sb_find_site($siteId);

    if (!$site) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    $currentUserId = (int)$USER->GetID();

    $bitrixGroupId = (int)(
        $site['bitrixGroupId']
        ?? $site['bitrix_group_id']
        ?? 0
    );

    if ($bitrixGroupId > 0) {
        sb_json_ok([
            'site' => $site,
            'bitrixGroupId' => $bitrixGroupId,
            'created' => false,
            'handler' => 'site',
            'action' => 'site.ensureGroup',
            'file' => __FILE__,
        ]);
    }

    if (!class_exists('SiteBitrixGroupService')) {
        sb_json_error('SiteBitrixGroupService.php не подключен', 500, [
            'handler' => 'site',
            'action' => 'site.ensureGroup',
            'file' => __FILE__,
        ]);
    }

    try {
        $bitrixGroupId = SiteBitrixGroupService::createForSite([
            'id' => $siteId,
            'name' => (string)($site['name'] ?? ('Сайт #' . $siteId)),
        ], $currentUserId);

        if ($bitrixGroupId <= 0) {
            throw new RuntimeException('BITRIX_GROUP_CREATE_EMPTY_ID');
        }

        sb_db_execute("
            UPDATE sitebuilder.site
            SET
                bitrix_group_id = :bitrix_group_id,
                bitrix_group_created_by = :created_by,
                bitrix_group_created_at = now()
            WHERE id = :site_id
        ", [
            ':bitrix_group_id' => $bitrixGroupId,
            ':created_by' => $currentUserId,
            ':site_id' => $siteId,
        ]);

        $site = sb_find_site($siteId);

        sb_json_ok([
            'site' => $site,
            'bitrixGroupId' => $bitrixGroupId,
            'created' => true,
            'handler' => 'site',
            'action' => 'site.ensureGroup',
            'file' => __FILE__,
        ]);
    } catch (Throwable $e) {
        sb_json_error($e->getMessage(), 500, [
            'handler' => 'site',
            'action' => 'site.ensureGroup',
            'file' => __FILE__,
        ]);
    }
}


---

3. В editor.php добавь кнопку в блок группы

В блоке «Группа Битрикс24 и права» найди:

<div class="sb-editor-inspector-actions">
    <a class="sb-btn sb-btn-light" href="#" target="_blank" id="openBitrixGroupBtn">Открыть группу</a>
    <button class="sb-btn sb-btn-primary" type="button" id="syncAccessBtn">Синхронизировать права</button>
</div>

Замени на:

<div class="sb-editor-inspector-actions">
    <button class="sb-btn sb-btn-primary" type="button" id="ensureBitrixGroupBtn">Создать группу Битрикс24</button>
    <a class="sb-btn sb-btn-light" href="#" target="_blank" id="openBitrixGroupBtn">Открыть группу</a>
    <button class="sb-btn sb-btn-primary" type="button" id="syncAccessBtn">Синхронизировать права</button>
</div>


---

4. В JS обнови renderBitrixGroupPanel()

Найди внутри функции:

var openBtn = document.getElementById('openBitrixGroupBtn');
var syncBtn = document.getElementById('syncAccessBtn');

Сделай так:

var openBtn = document.getElementById('openBitrixGroupBtn');
var syncBtn = document.getElementById('syncAccessBtn');
var ensureBtn = document.getElementById('ensureBitrixGroupBtn');

И ниже добавь:

if (ensureBtn) {
    if (groupId > 0) {
        ensureBtn.classList.add('sb-hidden');
    } else {
        ensureBtn.classList.remove('sb-hidden');
    }
}

Полная логика кнопок должна быть такой:

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

    if (groupId > 0) {
        syncBtn.classList.remove('sb-hidden');
    } else {
        syncBtn.classList.add('sb-hidden');
    }
}

if (ensureBtn) {
    if (groupId > 0) {
        ensureBtn.classList.add('sb-hidden');
    } else {
        ensureBtn.classList.remove('sb-hidden');
    }
}


---

5. Добавь JS-функцию создания группы

В editor.php внутри <script>, рядом с syncAccessFromBitrixGroup(), добавь:

async function ensureBitrixGroup() {
    var resultNode = document.getElementById('syncAccessResult');
    var btn = document.getElementById('ensureBitrixGroupBtn');

    if (resultNode) {
        resultNode.classList.remove('sb-hidden', 'is-success', 'is-error');
        resultNode.textContent = 'Создание группы Битрикс24...';
    }

    if (btn) {
        btn.disabled = true;
    }

    try {
        var res = await api('site.ensureGroup', {
            siteId: siteId
        });

        state.site = res.site || state.site;

        if (resultNode) {
            resultNode.classList.add('is-success');
            resultNode.textContent =
                res.created
                    ? 'Группа Битрикс24 создана. ID группы: ' + Number(res.bitrixGroupId || 0)
                    : 'Группа уже была создана. ID группы: ' + Number(res.bitrixGroupId || 0);
        }

        await loadSite();
    } catch (e) {
        if (resultNode) {
            resultNode.classList.add('is-error');
            resultNode.textContent = 'Ошибка создания группы: ' + ((e && (e.error || e.message)) || 'UNKNOWN_ERROR');
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

Внизу, рядом с обработчиком syncAccessBtn, добавь:

var ensureBitrixGroupBtn = document.getElementById('ensureBitrixGroupBtn');
if (ensureBitrixGroupBtn) {
    ensureBitrixGroupBtn.addEventListener('click', ensureBitrixGroup);
}


---

Проверка

Открой старый сайт, у которого:

bitrixGroupId = 0

В правом блоке должна появиться кнопка:

Создать группу Битрикс24

После нажатия должен вернуться ответ:

{
  "ok": true,
  "bitrixGroupId": 3,
  "created": true
}

После этого кнопка «Создать группу Битрикс24» скроется, а появятся:

Открыть группу
Синхронизировать права