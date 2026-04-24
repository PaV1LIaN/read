Да, сначала сделаем безопасную кнопку «Удалить сайт» в редакторе.

На первом этапе удаляем только сайт из SiteBuilder: страницы, блоки, меню, доступы, шаблоны, layout.
Группу Битрикс24 и файлы диска пока не удаляем автоматически, чтобы случайно не потерять документы. Это лучше сделать отдельной кнопкой позже.


---

1. Добавь кнопку в верхнюю панель

В editor.php найди блок:

<div class="sb-editor-topline-actions">
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/public.php?siteId=<?= (int)$siteId ?>" target="_blank">Открыть публичную</a>
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/layout.php?siteId=<?= (int)$siteId ?>">Layout</a>
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
</div>

Замени на:

<div class="sb-editor-topline-actions">
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/public.php?siteId=<?= (int)$siteId ?>" target="_blank">Открыть публичную</a>
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/layout.php?siteId=<?= (int)$siteId ?>">Layout</a>
    <a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/menu.php?siteId=<?= (int)$siteId ?>">Меню</a>
    <button class="sb-btn sb-btn-danger sb-btn-small" type="button" id="deleteSiteBtn">Удалить сайт</button>
</div>


---

2. Добавь JS-функцию удаления сайта

Внутри <script>, рядом с другими async function, например после loadSite(), добавь:

async function deleteCurrentSite() {
    var siteName = state.site && state.site.name ? state.site.name : ('siteId ' + siteId);

    var firstConfirm = confirm(
        'Удалить сайт "' + siteName + '"?\n\n' +
        'Будут удалены страницы, блоки, меню, доступы, шаблоны и layout внутри SiteBuilder.\n' +
        'Группа Битрикс24 и файлы диска сейчас не удаляются автоматически.'
    );

    if (!firstConfirm) {
        return;
    }

    var secondConfirm = confirm(
        'Подтверди удаление ещё раз.\n\n' +
        'Это действие нельзя будет отменить через интерфейс SiteBuilder.'
    );

    if (!secondConfirm) {
        return;
    }

    try {
        await api('site.delete', {
            id: siteId
        });

        alert('Сайт удалён');

        window.location.href = BASE_PATH + '/index.php';
    } catch (e) {
        var message = (e && (e.error || e.message)) ? (e.error || e.message) : 'UNKNOWN_ERROR';
        alert('Не удалось удалить сайт: ' + message);
    }
}


---

3. Добавь обработчик кнопки

Внизу, рядом с остальными addEventListener, после блока:

document.getElementById('moveBlockDownBtn').addEventListener('click', function () { moveBlock('down'); });

добавь:

var deleteSiteBtn = document.getElementById('deleteSiteBtn');
if (deleteSiteBtn) {
    deleteSiteBtn.addEventListener('click', deleteCurrentSite);
}


---

4. Проверка

Открой редактор сайта и нажми «Удалить сайт».

После подтверждения должен уйти запрос:

action=site.delete
id=<siteId>

Если пользователь владелец сайта, API вернёт:

{
  "ok": true,
  "handler": "site"
}

и тебя перекинет на:

/local/sitebuilder/index.php

Если будет ошибка, скорее всего это будет OWNER_REQUIRED или ACCESS_DENIED, потому что серверная проверка уже стоит здесь:

sb_require_owner($id);