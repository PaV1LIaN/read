Дальше добавляем блок шаблонов на главную страницу SiteBuilder:

/local/sitebuilder/index.php

Задача этого шага:

1. Показать список шаблонов.
2. Дать админу Битрикса создать сайт из шаблона.
3. Дать админу Битрикса удалить шаблон.


---

1. index.php — PHP-переменная админа

В начале index.php, после подключения Битрикса и global $USER;, добавь:

$isBitrixAdmin = is_object($USER)
    && method_exists($USER, 'IsAdmin')
    && $USER->IsAdmin();

Если global $USER; уже есть, просто добавь ниже него.


---

2. index.php — HTML-блок шаблонов

Вставь этот блок после блока со списком сайтов или перед ним, где тебе визуально удобнее.

<?php if ($isBitrixAdmin): ?>
    <section class="sb-panel sb-templates-panel">
        <div class="sb-panel-head">
            <div>
                <h2 class="sb-panel-title">Шаблоны сайтов</h2>
                <div class="sb-panel-subtitle">
                    Создавайте новые сайты на основе сохранённых шаблонов.
                </div>
            </div>

            <button class="sb-btn sb-btn-light sb-btn-small" type="button" id="reloadTemplatesBtn">
                Обновить
            </button>
        </div>

        <div class="sb-template-create-note">
            Создание, удаление и использование шаблонов доступно только администратору Битрикса.
        </div>

        <div id="templatesMessage" class="sb-templates-message" hidden></div>

        <div id="templatesList" class="sb-templates-list">
            <div class="sb-empty">Загрузка шаблонов...</div>
        </div>
    </section>

    <div class="sb-template-site-modal" id="createSiteFromTemplateModal" hidden>
        <div class="sb-template-site-modal__backdrop" data-close-create-site-template-modal></div>

        <div class="sb-template-site-modal__dialog">
            <div class="sb-template-site-modal__head">
                <div>
                    <h2 class="sb-template-site-modal__title">Создать сайт из шаблона</h2>
                    <p class="sb-template-site-modal__subtitle" id="createSiteTemplateName">
                        Выберите название нового сайта.
                    </p>
                </div>

                <button class="sb-template-site-modal__close" type="button" data-close-create-site-template-modal>
                    ×
                </button>
            </div>

            <div class="sb-template-site-modal__body">
                <input type="hidden" id="createSiteTemplateId" value="0">

                <div class="sb-field">
                    <label for="createSiteNameInput">Название сайта</label>
                    <input class="sb-input" type="text" id="createSiteNameInput" placeholder="Например: Новый портал">
                </div>

                <div class="sb-field" style="margin-top:12px;">
                    <label for="createSiteSlugInput">Символьный код</label>
                    <input class="sb-input" type="text" id="createSiteSlugInput" placeholder="new-portal">
                </div>

                <div class="sb-template-site-note">
                    Если оставить символьный код пустым, он будет создан автоматически.
                </div>

                <div id="createSiteFromTemplateMessage" class="sb-templates-message" hidden></div>
            </div>

            <div class="sb-template-site-modal__footer">
                <button class="sb-btn sb-btn-light" type="button" data-close-create-site-template-modal>
                    Отмена
                </button>

                <button class="sb-btn sb-btn-primary" type="button" id="createSiteFromTemplateBtn">
                    Создать сайт
                </button>
            </div>
        </div>
    </div>
<?php endif; ?>


---

3. index.php — JS для шаблонов

В самый низ страницы, перед </body> или внутри существующего <script>, добавь:

<?php if ($isBitrixAdmin): ?>
<script>
(function () {
    var templates = [];
    var selectedTemplateId = 0;

    function templateApi(action, data) {
        data = data || {};

        var body = new URLSearchParams();
        body.append('action', action);
        body.append('sessid', getSessid());

        Object.keys(data).forEach(function (key) {
            body.append(key, data[key]);
        });

        return fetch('/local/sitebuilder/api/index.php', {
            method: 'POST',
            credentials: 'same-origin',
            headers: {
                'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
            },
            body: body.toString()
        }).then(function (response) {
            return response.json();
        }).then(function (json) {
            if (!json || json.ok !== true) {
                var message = json && (json.message || json.error)
                    ? (json.message || json.error)
                    : 'UNKNOWN_ERROR';

                throw new Error(message);
            }

            return json.data || {};
        });
    }

    function getSessid() {
        if (window.BX && typeof BX.bitrix_sessid === 'function') {
            return BX.bitrix_sessid();
        }

        return '';
    }

    function escapeHtml(value) {
        return String(value == null ? '' : value)
            .replace(/&/g, '&amp;')
            .replace(/</g, '&lt;')
            .replace(/>/g, '&gt;')
            .replace(/"/g, '&quot;')
            .replace(/'/g, '&#039;');
    }

    function setTemplatesMessage(text, type) {
        var node = document.getElementById('templatesMessage');

        if (!node) {
            return;
        }

        node.hidden = !text;
        node.textContent = text || '';
        node.className = 'sb-templates-message' + (type ? ' is-' + type : '');
    }

    function setCreateSiteMessage(text, type) {
        var node = document.getElementById('createSiteFromTemplateMessage');

        if (!node) {
            return;
        }

        node.hidden = !text;
        node.textContent = text || '';
        node.className = 'sb-templates-message' + (type ? ' is-' + type : '');
    }

    async function loadTemplates() {
        var list = document.getElementById('templatesList');

        if (!list) {
            return;
        }

        list.innerHTML = '<div class="sb-empty">Загрузка шаблонов...</div>';
        setTemplatesMessage('', '');

        try {
            var data = await templateApi('template.list');

            templates = Array.isArray(data.templates) ? data.templates : [];

            renderTemplates();
        } catch (e) {
            list.innerHTML = '<div class="sb-empty">Не удалось загрузить шаблоны</div>';
            setTemplatesMessage('Ошибка загрузки шаблонов: ' + e.message, 'error');
        }
    }

    function renderTemplates() {
        var list = document.getElementById('templatesList');

        if (!list) {
            return;
        }

        if (!templates.length) {
            list.innerHTML = ''
                + '<div class="sb-template-empty">'
                + '    <div class="sb-template-empty__icon">🧩</div>'
                + '    <strong>Шаблонов пока нет</strong>'
                + '    <span>Открой сайт в редакторе и нажми “Сохранить как шаблон”.</span>'
                + '</div>';
            return;
        }

        list.innerHTML = templates.map(function (template) {
            var title = template.name || 'Шаблон #' + template.id;
            var description = template.description || 'Описание не заполнено';
            var sourceSiteName = template.sourceSiteName || '—';
            var pagesCount = Number(template.pagesCount || 0);
            var blocksCount = Number(template.blocksCount || 0);
            var updatedAt = template.updatedAt || template.createdAt || '';

            return ''
                + '<article class="sb-template-card" data-template-id="' + Number(template.id || 0) + '">'
                + '    <div class="sb-template-card__top">'
                + '        <div class="sb-template-card__icon">🧩</div>'
                + '        <div class="sb-template-card__main">'
                + '            <div class="sb-template-card__title">' + escapeHtml(title) + '</div>'
                + '            <div class="sb-template-card__desc">' + escapeHtml(description) + '</div>'
                + '        </div>'
                + '    </div>'
                + ''
                + '    <div class="sb-template-card__meta">'
                + '        <span>Источник: ' + escapeHtml(sourceSiteName) + '</span>'
                + '        <span>' + pagesCount + ' стр.</span>'
                + '        <span>' + blocksCount + ' блок.</span>'
                + (updatedAt ? '<span>' + escapeHtml(updatedAt) + '</span>' : '')
                + '    </div>'
                + ''
                + '    <div class="sb-template-card__actions">'
                + '        <button class="sb-btn sb-btn-primary sb-btn-small" type="button" data-template-action="create-site" data-template-id="' + Number(template.id || 0) + '">'
                + '            Создать сайт'
                + '        </button>'
                + '        <button class="sb-btn sb-btn-danger sb-btn-small" type="button" data-template-action="delete" data-template-id="' + Number(template.id || 0) + '">'
                + '            Удалить'
                + '        </button>'
                + '    </div>'
                + '</article>';
        }).join('');
    }

    function findTemplate(templateId) {
        templateId = Number(templateId || 0);

        for (var i = 0; i < templates.length; i++) {
            if (Number(templates[i].id || 0) === templateId) {
                return templates[i];
            }
        }

        return null;
    }

    function openCreateSiteModal(templateId) {
        var template = findTemplate(templateId);

        if (!template) {
            alert('Шаблон не найден');
            return;
        }

        selectedTemplateId = templateId;

        var modal = document.getElementById('createSiteFromTemplateModal');
        var idInput = document.getElementById('createSiteTemplateId');
        var nameInput = document.getElementById('createSiteNameInput');
        var slugInput = document.getElementById('createSiteSlugInput');
        var title = document.getElementById('createSiteTemplateName');

        if (!modal || !idInput || !nameInput) {
            return;
        }

        idInput.value = String(templateId);
        nameInput.value = template.name ? String(template.name) : '';
        if (slugInput) {
            slugInput.value = '';
        }

        if (title) {
            title.textContent = 'Шаблон: ' + (template.name || ('#' + templateId));
        }

        setCreateSiteMessage('', '');

        modal.hidden = false;

        setTimeout(function () {
            nameInput.focus();
            nameInput.select();
        }, 50);
    }

    function closeCreateSiteModal() {
        var modal = document.getElementById('createSiteFromTemplateModal');

        if (modal) {
            modal.hidden = true;
        }

        selectedTemplateId = 0;
    }

    async function createSiteFromTemplate() {
        var templateIdInput = document.getElementById('createSiteTemplateId');
        var nameInput = document.getElementById('createSiteNameInput');
        var slugInput = document.getElementById('createSiteSlugInput');
        var btn = document.getElementById('createSiteFromTemplateBtn');

        var templateId = Number(templateIdInput && templateIdInput.value ? templateIdInput.value : selectedTemplateId);
        var name = nameInput ? String(nameInput.value || '').trim() : '';
        var slug = slugInput ? String(slugInput.value || '').trim() : '';

        if (!templateId) {
            alert('Шаблон не выбран');
            return;
        }

        if (!name) {
            alert('Введите название сайта');
            if (nameInput) {
                nameInput.focus();
            }
            return;
        }

        if (btn) {
            btn.disabled = true;
            btn.textContent = 'Создаю...';
        }

        setCreateSiteMessage('Создаю сайт...', 'info');

        try {
            var data = await templateApi('template.createSite', {
                templateId: templateId,
                name: name,
                slug: slug
            });

            var site = data.site || {};
            var siteId = Number(site.id || 0);

            setCreateSiteMessage('Сайт создан', 'success');

            if (siteId > 0) {
                setTimeout(function () {
                    window.location.href = '/local/sitebuilder/editor.php?siteId=' + siteId;
                }, 400);
            } else {
                closeCreateSiteModal();
                window.location.reload();
            }
        } catch (e) {
            setCreateSiteMessage('Не удалось создать сайт: ' + e.message, 'error');
        } finally {
            if (btn) {
                btn.disabled = false;
                btn.textContent = 'Создать сайт';
            }
        }
    }

    async function deleteTemplate(templateId) {
        var template = findTemplate(templateId);
        var name = template && template.name ? template.name : ('#' + templateId);

        if (!confirm('Удалить шаблон "' + name + '"?')) {
            return;
        }

        setTemplatesMessage('Удаляю шаблон...', 'info');

        try {
            await templateApi('template.delete', {
                templateId: templateId
            });

            setTemplatesMessage('Шаблон удалён', 'success');
            await loadTemplates();
        } catch (e) {
            setTemplatesMessage('Не удалось удалить шаблон: ' + e.message, 'error');
        }
    }

    document.addEventListener('click', function (e) {
        var actionBtn = e.target.closest('[data-template-action]');
        if (actionBtn) {
            var action = actionBtn.getAttribute('data-template-action');
            var templateId = Number(actionBtn.getAttribute('data-template-id') || 0);

            if (action === 'create-site') {
                openCreateSiteModal(templateId);
                return;
            }

            if (action === 'delete') {
                deleteTemplate(templateId);
                return;
            }
        }

        if (e.target.closest('[data-close-create-site-template-modal]')) {
            closeCreateSiteModal();
        }
    });

    var reloadBtn = document.getElementById('reloadTemplatesBtn');
    if (reloadBtn) {
        reloadBtn.addEventListener('click', loadTemplates);
    }

    var createBtn = document.getElementById('createSiteFromTemplateBtn');
    if (createBtn) {
        createBtn.addEventListener('click', createSiteFromTemplate);
    }

    document.addEventListener('DOMContentLoaded', loadTemplates);

    if (document.readyState === 'interactive' || document.readyState === 'complete') {
        loadTemplates();
    }
})();
</script>
<?php endif; ?>


---

4. CSS для index.php

В конец CSS-файла, который подключён на главной странице SiteBuilder, добавь этот блок.
Если отдельного CSS для главной нет, можно временно добавить в:

/local/sitebuilder/assets/admin/editor.css

/* =========================================================
   SITE TEMPLATES ON DASHBOARD
   ========================================================= */

.sb-templates-panel {
    margin-top: 18px;
}

.sb-template-create-note {
    margin: 12px 0 14px;
    padding: 10px 12px;
    border: 1px solid #dbeafe;
    border-radius: 12px;
    background: #eff6ff;
    color: #1e40af;
    font-size: 12px;
    line-height: 1.45;
}

.sb-templates-message {
    margin: 12px 0;
    padding: 10px 12px;
    border-radius: 12px;
    background: #f3f4f6;
    color: #374151;
    font-size: 13px;
}

.sb-templates-message.is-success {
    background: #dcfce7;
    color: #166534;
}

.sb-templates-message.is-error {
    background: #fee2e2;
    color: #991b1b;
}

.sb-templates-message.is-info {
    background: #eef2ff;
    color: #3730a3;
}

.sb-templates-list {
    display: grid;
    grid-template-columns: repeat(auto-fill, minmax(280px, 1fr));
    gap: 12px;
}

.sb-template-card {
    padding: 16px;
    border: 1px solid #e5e7eb;
    border-radius: 18px;
    background: #fff;
    box-shadow: 0 10px 26px rgba(15, 23, 42, .04);
}

.sb-template-card__top {
    display: flex;
    gap: 12px;
    align-items: flex-start;
}

.sb-template-card__icon {
    width: 42px;
    height: 42px;
    flex: 0 0 42px;
    border-radius: 14px;
    background: #eef2ff;
    color: #3730a3;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 22px;
}

.sb-template-card__main {
    min-width: 0;
}

.sb-template-card__title {
    color: #111827;
    font-size: 15px;
    font-weight: 900;
    line-height: 1.25;
}

.sb-template-card__desc {
    margin-top: 4px;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.4;
}

.sb-template-card__meta {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-top: 14px;
}

.sb-template-card__meta span {
    display: inline-flex;
    align-items: center;
    min-height: 24px;
    padding: 0 8px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #4b5563;
    font-size: 12px;
    font-weight: 700;
}

.sb-template-card__actions {
    display: flex;
    gap: 8px;
    flex-wrap: wrap;
    margin-top: 14px;
}

.sb-template-empty {
    grid-column: 1 / -1;
    min-height: 160px;
    padding: 28px;
    border: 1px dashed #cbd5e1;
    border-radius: 18px;
    background: #fff;
    text-align: center;
    color: #6b7280;
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    gap: 8px;
}

.sb-template-empty__icon {
    width: 56px;
    height: 56px;
    border-radius: 20px;
    background: #eef2ff;
    color: #3730a3;
    display: flex;
    align-items: center;
    justify-content: center;
    font-size: 28px;
}

.sb-template-empty strong {
    color: #111827;
    font-size: 16px;
}

.sb-template-empty span {
    color: #6b7280;
    font-size: 13px;
}

/* Modal create site from template */

.sb-template-site-modal[hidden] {
    display: none !important;
}

.sb-template-site-modal {
    position: fixed;
    inset: 0;
    z-index: 10000;
    display: flex;
    align-items: center;
    justify-content: center;
    padding: 24px;
}

.sb-template-site-modal__backdrop {
    position: absolute;
    inset: 0;
    background: rgba(15, 23, 42, .48);
    backdrop-filter: blur(5px);
}

.sb-template-site-modal__dialog {
    position: relative;
    width: min(520px, 100%);
    overflow: hidden;
    border: 1px solid #e5e7eb;
    border-radius: 22px;
    background: #fff;
    box-shadow: 0 28px 80px rgba(15, 23, 42, .30);
}

.sb-template-site-modal__head {
    display: flex;
    justify-content: space-between;
    gap: 16px;
    padding: 20px 22px;
    border-bottom: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-template-site-modal__title {
    margin: 0;
    font-size: 20px;
    font-weight: 900;
    color: #111827;
}

.sb-template-site-modal__subtitle {
    margin: 6px 0 0;
    color: #6b7280;
    font-size: 13px;
    line-height: 1.45;
}

.sb-template-site-modal__close {
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

.sb-template-site-modal__close:hover {
    background: #f3f4f6;
    color: #111827;
}

.sb-template-site-modal__body {
    padding: 22px;
}

.sb-template-site-modal__footer {
    display: flex;
    justify-content: flex-end;
    gap: 10px;
    padding: 16px 22px;
    border-top: 1px solid #eef2f7;
    background: #f9fafb;
}

.sb-template-site-note {
    margin-top: 12px;
    padding: 10px 12px;
    border: 1px solid #dbeafe;
    border-radius: 12px;
    background: #eff6ff;
    color: #1e40af;
    font-size: 12px;
    line-height: 1.45;
}


---

После этого на главной странице появится блок “Шаблоны сайтов”.
Если шаблон уже создан из редактора, он должен отобразиться карточкой.