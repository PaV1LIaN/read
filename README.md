Да, так правильнее:

1. Создать сайт показываем только администратору Битрикс.


2. Нижний блок «Создать сайт» убираем.


3. Верхняя кнопка «Создать сайт» открывает модальное окно.


4. На сервере тоже закрываем site.create, чтобы обычный пользователь не смог создать сайт через API.




---

1. В index.php после $basePath добавь

$canCreateSite = $USER->IsAdmin();

Было:

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');

Должно стать:

$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$canCreateSite = $USER->IsAdmin();


---

2. Верхнюю кнопку «Создать сайт» оберни проверкой

Было:

<button class="sb-btn sb-btn-primary" type="button" id="createSiteQuickBtn">Создать сайт</button>

Замени на:

<?php if ($canCreateSite): ?>
    <button class="sb-btn sb-btn-primary" type="button" id="createSiteQuickBtn">Создать сайт</button>
<?php endif; ?>


---

3. Удали нижний блок «Создать сайт»

Удали полностью этот блок:

<section class="sb-dash-card">
    <div class="sb-dash-card__head">
        <h2>Создать сайт</h2>
    </div>
    <div class="sb-form-row align-end">
        <div class="sb-field">
            <label for="siteName">Название сайта</label>
            <input class="sb-input" type="text" id="siteName" placeholder="Например: Корпоративный портал">
        </div>
        <div class="sb-field">
            <label for="siteSlug">Slug</label>
            <input class="sb-input" type="text" id="siteSlug" placeholder="Например: corp-portal">
        </div>
        <button type="button" class="sb-btn sb-btn-primary" id="createSiteBtn">Создать</button>
    </div>
</section>


---

4. Добавь модальное окно перед блоком «Отладка»

Перед этим блоком:

<section class="sb-dash-card">
    <div class="sb-dash-card__head">
        <h2>Отладка</h2>
    </div>
    <div id="output" class="sb-output">Здесь будут ответы API...</div>
</section>

вставь:

<?php if ($canCreateSite): ?>
    <div class="sb-modal" id="createSiteModal" hidden>
        <div class="sb-modal__backdrop" data-close-create-modal></div>

        <div class="sb-modal__dialog">
            <div class="sb-modal__head">
                <div>
                    <h2 class="sb-modal__title">Создать сайт</h2>
                    <p class="sb-modal__subtitle">Новый сайт автоматически получит рабочую группу Битрикс24.</p>
                </div>

                <button class="sb-modal__close" type="button" data-close-create-modal>×</button>
            </div>

            <div class="sb-modal__body">
                <div class="sb-field">
                    <label for="siteName">Название сайта</label>
                    <input class="sb-input" type="text" id="siteName" placeholder="Например: Корпоративный портал">
                </div>

                <div class="sb-field" style="margin-top:12px;">
                    <label for="siteSlug">Slug</label>
                    <input class="sb-input" type="text" id="siteSlug" placeholder="Например: corp-portal">
                </div>
            </div>

            <div class="sb-modal__footer">
                <button class="sb-btn sb-btn-light" type="button" data-close-create-modal>Отмена</button>
                <button class="sb-btn sb-btn-primary" type="button" id="createSiteBtn">Создать</button>
            </div>
        </div>
    </div>
<?php endif; ?>


---

5. В <style> добавь стили модалки

Внутрь <style> добавь:

.sb-modal[hidden] {
    display: none !important;
}

.sb-modal {
    position: fixed;
    inset: 0;
    z-index: 10000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
}

.sb-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(15, 23, 42, 0.45);
    backdrop-filter: blur(4px);
}

.sb-modal__dialog {
    position: relative;
    width: min(520px, 100%);
    background: #fff;
    border-radius: 20px;
    box-shadow: 0 24px 70px rgba(15, 23, 42, 0.28);
    overflow: hidden;
    border: 1px solid #e5e7eb;
}

.sb-modal__head {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    padding: 20px 22px;
    border-bottom: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-modal__title {
    margin: 0;
    font-size: 20px;
    font-weight: 800;
    color: #111827;
}

.sb-modal__subtitle {
    margin: 6px 0 0;
    font-size: 13px;
    color: #6b7280;
    line-height: 1.45;
}

.sb-modal__close {
    width: 34px;
    height: 34px;
    border: 1px solid #e5e7eb;
    border-radius: 10px;
    background: #fff;
    cursor: pointer;
    font-size: 22px;
    line-height: 1;
    color: #6b7280;
}

.sb-modal__close:hover {
    background: #f3f4f6;
    color: #111827;
}

.sb-modal__body {
    padding: 22px;
}

.sb-modal__footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 22px;
    border-top: 1px solid #eef2f7;
    background: #f9fafb;
}


---

6. В JS добавь флаг администратора

После:

var API_URL = BASE_PATH + '/api.php';

добавь:

var CAN_CREATE_SITE = <?= $canCreateSite ? 'true' : 'false' ?>;


---

7. Добавь функции открытия/закрытия модалки

Рядом с другими функциями добавь:

function openCreateSiteModal() {
    if (!CAN_CREATE_SITE) {
        return;
    }

    var modal = document.getElementById('createSiteModal');
    if (!modal) {
        return;
    }

    modal.hidden = false;

    setTimeout(function () {
        var nameInput = document.getElementById('siteName');
        if (nameInput) {
            nameInput.focus();
        }
    }, 50);
}

function closeCreateSiteModal() {
    var modal = document.getElementById('createSiteModal');
    if (!modal) {
        return;
    }

    modal.hidden = true;
}


---

8. Обнови createSite()

Замени начало функции createSite() на:

function createSite() {
    if (!CAN_CREATE_SITE) {
        alert('Создавать сайты может только администратор Битрикс24');
        return;
    }

    var nameInput = document.getElementById('siteName');
    var slugInput = document.getElementById('siteSlug');

    if (!nameInput || !slugInput) {
        alert('Форма создания сайта не найдена');
        return;
    }

А внутри успешного ответа после очистки полей добавь закрытие модалки:

nameInput.value = '';
slugInput.value = '';
closeCreateSiteModal();
loadSites();


---

9. Замени обработчики кнопок создания

Было:

document.getElementById('createSiteBtn').addEventListener('click', createSite);

document.getElementById('createSiteQuickBtn').addEventListener('click', function () {
    document.getElementById('siteName').focus();
    window.scrollTo({
        top: document.body.scrollHeight,
        behavior: 'smooth'
    });
});

Замени на:

var createSiteBtn = document.getElementById('createSiteBtn');
if (createSiteBtn) {
    createSiteBtn.addEventListener('click', createSite);
}

var createSiteQuickBtn = document.getElementById('createSiteQuickBtn');
if (createSiteQuickBtn) {
    createSiteQuickBtn.addEventListener('click', openCreateSiteModal);
}

document.querySelectorAll('[data-close-create-modal]').forEach(function (btn) {
    btn.addEventListener('click', closeCreateSiteModal);
});

document.addEventListener('keydown', function (e) {
    if (e.key === 'Escape') {
        closeCreateSiteModal();
    }
});


---

10. В site.php закрой создание сайта на сервере

В файле:

/local/sitebuilder/api/handlers/site.php

внутри блока:

if ($action === 'site.create') {

сразу после открытия добавь:

if (!$USER->IsAdmin()) {
    sb_json_error('BITRIX_ADMIN_REQUIRED', 403);
}

Должно быть так:

if ($action === 'site.create') {
    if (!$USER->IsAdmin()) {
        sb_json_error('BITRIX_ADMIN_REQUIRED', 403);
    }

    $name = trim((string)($_POST['name'] ?? ''));
    if ($name === '') {
        sb_json_error('NAME_REQUIRED', 422);
    }

    ...
}

После этого обычный пользователь не увидит кнопку создания и не сможет создать сайт через API.