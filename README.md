Дальше делаем интерфейс сохранения шаблона в редакторе.

Сейчас трогаем 2 файла:

/local/sitebuilder/editor.php
/local/sitebuilder/assets/admin/editor.css


---

1. editor.php — кнопка “Сохранить как шаблон”

Найди в верхней панели ссылку на настройки:

<a class="sb-btn sb-btn-light sb-btn-small" href="<?= htmlspecialchars($basePath, ENT_QUOTES | ENT_SUBSTITUTE, 'UTF-8') ?>/settings.php?siteId=<?= (int)$siteId ?>">
    Настройки
</a>

Сразу после неё вставь:

<?php if ($USER->IsAdmin()): ?>
    <button class="sb-btn sb-btn-primary sb-btn-small" type="button" id="saveAsTemplateBtn">
        Сохранить как шаблон
    </button>
<?php endif; ?>


---

2. editor.php — модальное окно

Найди конец основной HTML-разметки перед строками:

<script src="/bitrix/js/main/core/core.js"></script>
<script>

И перед ними вставь:

<?php if ($USER->IsAdmin()): ?>
    <div class="sb-template-modal" id="saveTemplateModal" hidden>
        <div class="sb-template-modal__backdrop" data-close-template-modal></div>

        <div class="sb-template-modal__dialog">
            <div class="sb-template-modal__head">
                <div>
                    <h2 class="sb-template-modal__title">Сохранить сайт как шаблон</h2>
                    <p class="sb-template-modal__subtitle">
                        Шаблон сохранит страницы, вложенность, блоки, layout, меню и оформление. Файлы диска не копируются.
                    </p>
                </div>

                <button class="sb-template-modal__close" type="button" data-close-template-modal>×</button>
            </div>

            <div class="sb-template-modal__body">
                <div class="sb-field">
                    <label for="templateNameInput">Название шаблона</label>
                    <input class="sb-input" type="text" id="templateNameInput" placeholder="Например: Корпоративный портал">
                </div>

                <div class="sb-field" style="margin-top:12px;">
                    <label for="templateDescriptionInput">Описание</label>
                    <textarea class="sb-input" id="templateDescriptionInput" rows="4" placeholder="Кратко опиши, для каких сайтов подходит этот шаблон"></textarea>
                </div>

                <div class="sb-template-note">
                    Создание, изменение и удаление шаблонов доступно только администратору Битрикса.
                </div>

                <div id="templateMessage" class="sb-template-message" hidden></div>
            </div>

            <div class="sb-template-modal__footer">
                <button class="sb-btn sb-btn-light" type="button" data-close-template-modal>Отмена</button>
                <button class="sb-btn sb-btn-primary" type="button" id="createTemplateBtn">Создать шаблон</button>
            </div>
        </div>
    </div>
<?php endif; ?>


---

3. editor.php — JS-функции

Внутри <script>, ниже функций, но перед функцией deleteSite(), вставь:

function openTemplateModal() {
    if (!IS_BITRIX_ADMIN) {
        alert('Создавать шаблоны может только администратор Битрикса');
        return;
    }

    var modal = document.getElementById('saveTemplateModal');
    if (!modal) return;

    var nameInput = document.getElementById('templateNameInput');
    var descInput = document.getElementById('templateDescriptionInput');
    var message = document.getElementById('templateMessage');

    if (nameInput && !nameInput.value) {
        var siteName = state.site && state.site.name ? state.site.name : 'Сайт';
        nameInput.value = siteName;
    }

    if (descInput && !descInput.value) {
        descInput.value = '';
    }

    if (message) {
        message.hidden = true;
        message.textContent = '';
        message.className = 'sb-template-message';
    }

    modal.hidden = false;

    setTimeout(function () {
        if (nameInput) {
            nameInput.focus();
            nameInput.select();
        }
    }, 50);
}

function closeTemplateModal() {
    var modal = document.getElementById('saveTemplateModal');
    if (!modal) return;

    modal.hidden = true;
}

function setTemplateMessage(text, type) {
    var message = document.getElementById('templateMessage');
    if (!message) return;

    message.hidden = !text;
    message.textContent = text || '';
    message.className = 'sb-template-message' + (type ? ' is-' + type : '');
}

async function createTemplateFromSite() {
    if (!IS_BITRIX_ADMIN) {
        alert('Создавать шаблоны может только администратор Битрикса');
        return;
    }

    var nameInput = document.getElementById('templateNameInput');
    var descInput = document.getElementById('templateDescriptionInput');
    var btn = document.getElementById('createTemplateBtn');

    var name = nameInput ? String(nameInput.value || '').trim() : '';
    var description = descInput ? String(descInput.value || '').trim() : '';

    if (!name) {
        alert('Введите название шаблона');
        if (nameInput) nameInput.focus();
        return;
    }

    if (btn) {
        btn.disabled = true;
        btn.textContent = 'Создаю...';
    }

    setTemplateMessage('Создаю шаблон...', 'info');

    try {
        await api('template.createFromSite', {
            siteId: siteId,
            name: name,
            description: description
        });

        setTemplateMessage('Шаблон создан', 'success');

        setTimeout(function () {
            closeTemplateModal();
        }, 350);
    } catch (e) {
        var message = e && (e.message || e.error) ? (e.message || e.error) : 'UNKNOWN_ERROR';
        setTemplateMessage('Не удалось создать шаблон: ' + message, 'error');
    } finally {
        if (btn) {
            btn.disabled = false;
            btn.textContent = 'Создать шаблон';
        }
    }
}


---

4. editor.php — обработчики кнопок

Найди место, где подключаются обработчики кнопок, рядом с:

if (deleteSiteBtn) {
    deleteSiteBtn.addEventListener('click', deleteSite);
}

Сразу после этого вставь:

var saveAsTemplateBtn = document.getElementById('saveAsTemplateBtn');
if (saveAsTemplateBtn) {
    saveAsTemplateBtn.addEventListener('click', openTemplateModal);
}

var createTemplateBtn = document.getElementById('createTemplateBtn');
if (createTemplateBtn) {
    createTemplateBtn.addEventListener('click', createTemplateFromSite);
}

document.querySelectorAll('[data-close-template-modal]').forEach(function (btn) {
    btn.addEventListener('click', closeTemplateModal);
});


---

5. editor.css

В конец файла:

/local/sitebuilder/assets/admin/editor.css

добавь:

/* =========================================================
   SITE TEMPLATES MODAL
   ========================================================= */

.sb-template-modal[hidden] {
    display: none !important;
}

.sb-template-modal {
    position: fixed;
    inset: 0;
    z-index: 10000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
}

.sb-template-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(15, 23, 42, .48);
    backdrop-filter: blur(5px);
}

.sb-template-modal__dialog {
    position: relative;
    width: min(560px, 100%);
    overflow: hidden;
    border: 1px solid #e5e7eb;
    border-radius: 22px;
    background: #fff;
    box-shadow: 0 28px 80px rgba(15, 23, 42, .30);
}

.sb-template-modal__head {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    padding: 20px 22px;
    border-bottom: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-template-modal__title {
    margin: 0;
    font-size: 20px;
    font-weight: 900;
    color: #111827;
}

.sb-template-modal__subtitle {
    margin: 6px 0 0;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.45;
}

.sb-template-modal__close {
    width: 34px;
    height: 34px;
    border: 1px solid #e5e7eb;
    border-radius: 11px;
    background: #fff;
    color: #6b7280;
    cursor: pointer;
    font-size: 22px;
    line-height: 1;
}

.sb-template-modal__close:hover {
    background: #f3f4f6;
    color: #111827;
}

.sb-template-modal__body {
    padding: 22px;
}

.sb-template-modal__footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 22px;
    border-top: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-template-note {
    margin-top: 12px;
    padding: 10px 12px;
    border: 1px solid #dbeafe;
    border-radius: 12px;
    background: #eff6ff;
    color: #1e40af;
    font-size: 12px;
    line-height: 1.45;
}

.sb-template-message {
    margin-top: 12px;
    padding: 10px 12px;
    border-radius: 12px;
    background: #f3f4f6;
    color: #374151;
    font-size: 13px;
}

.sb-template-message.is-success {
    background: #dcfce7;
    color: #166534;
}

.sb-template-message.is-error {
    background: #fee2e2;
    color: #991b1b;
}

.sb-template-message.is-info {
    background: #eef2ff;
    color: #3730a3;
}

После этого в редакторе у администратора Битрикса должна появиться кнопка “Сохранить как шаблон”. Следующим шагом дам код для index.php, чтобы на главной странице появился блок шаблонов и создание сайта из шаблона.