Дальше делаем **отображение сайтов по секциям на главной странице**.

Итог будет такой:

Для обычного пользователя:

```text
Информационные порталы
  321
  Информационный портал "Сервис Строка"

Без секции
  Другой сайт
```

Для администратора Битрикс24:

```text
ID | Сайт | Секция | Роль | Slug | ...
```

И будет возможность назначить сайт в секцию через выпадающий список.

---

## 1. В `index.php` в `<style>` добавь

```css
.sb-section-group {
    margin-bottom: 18px;
}

.sb-section-title {
    margin: 0 0 8px;
    font-size: 15px;
    font-weight: 800;
    color: #111827;
}

.sb-section-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-site-section-select {
    min-width: 180px;
    height: 32px;
    border: 1px solid #d1d5db;
    border-radius: 10px;
    padding: 0 8px;
    background: #fff;
    font-size: 12px;
}
```

---

## 2. В верхних кнопках добавь кнопку создания секции

Найди:

```php
<?php if ($canCreateSite): ?>
    <button class="sb-btn sb-btn-primary" type="button" id="createSiteQuickBtn">Создать сайт</button>
<?php endif; ?>
```

Замени на:

```php
<?php if ($canCreateSite): ?>
    <button class="sb-btn sb-btn-light" type="button" id="createSectionBtn">Создать секцию</button>
    <button class="sb-btn sb-btn-primary" type="button" id="createSiteQuickBtn">Создать сайт</button>
<?php endif; ?>
```

---

## 3. В таблицу администратора добавь колонку «Секция»

В `<thead>` для админа было:

```php
<th>ID</th>
<th>Сайт</th>
<th>Роль</th>
<th>Slug</th>
```

Сделай так:

```php
<th>ID</th>
<th>Сайт</th>
<th>Секция</th>
<th>Роль</th>
<th>Slug</th>
```

И везде, где для админа стоит `colspan="9"`, поменяй на `10`.

Например:

```php
<td colspan="<?= $isBitrixAdmin ? 10 : 2 ?>">Загрузка...</td>
```

---

## 4. Добавь модальное окно создания секции

Поставь рядом с модальным окном создания сайта:

```php
<?php if ($canCreateSite): ?>
    <div class="sb-modal" id="createSectionModal" hidden>
        <div class="sb-modal__backdrop" data-close-section-modal></div>

        <div class="sb-modal__dialog">
            <div class="sb-modal__head">
                <div>
                    <h2 class="sb-modal__title">Создать секцию</h2>
                    <p class="sb-modal__subtitle">
                        Секция нужна для группировки сайтов на главной странице.
                    </p>
                </div>

                <button class="sb-modal__close" type="button" data-close-section-modal>×</button>
            </div>

            <div class="sb-modal__body">
                <div class="sb-field">
                    <label for="sectionName">Название секции</label>
                    <input class="sb-input" type="text" id="sectionName" placeholder="Например: Информационные порталы">
                </div>

                <div class="sb-field" style="margin-top:12px;">
                    <label for="sectionSort">Сортировка</label>
                    <input class="sb-input" type="number" id="sectionSort" value="500">
                </div>
            </div>

            <div class="sb-modal__footer">
                <button class="sb-btn sb-btn-light" type="button" data-close-section-modal>Отмена</button>
                <button class="sb-btn sb-btn-primary" type="button" id="saveSectionBtn">Создать</button>
            </div>
        </div>
    </div>
<?php endif; ?>
```

---

## 5. В JS добавь массив секций

Найди:

```js
var allSites = [];
```

Ниже добавь:

```js
var allSections = [];
```

---

## 6. Добавь функции секций в JS

Вставь рядом с другими функциями:

```js
function getSectionName(sectionId) {
    sectionId = Number(sectionId || 0);

    if (sectionId <= 0) {
        return 'Без секции';
    }

    var section = allSections.find(function (s) {
        return Number(s.id || 0) === sectionId;
    });

    return section ? String(section.name || 'Без названия') : 'Без секции';
}

function renderSectionSelect(site) {
    var siteId = Number(site.id || 0);
    var currentSectionId = Number(site.sectionId || 0);

    var html = '<select class="sb-site-section-select" data-action="set-section" data-site-id="' + siteId + '">';
    html += '<option value="0">Без секции</option>';

    allSections.forEach(function (section) {
        var id = Number(section.id || 0);
        html += '<option value="' + id + '"' + (id === currentSectionId ? ' selected' : '') + '>'
            + escapeHtml(section.name || '')
            + '</option>';
    });

    html += '</select>';

    return html;
}

function loadSections(callback) {
    api('section.list', {}, function (res) {
        allSections = Array.isArray(res.sections) ? res.sections : [];

        if (typeof callback === 'function') {
            callback();
        }
    }, function () {
        allSections = [];

        if (typeof callback === 'function') {
            callback();
        }
    });
}

function groupSitesBySection(sites) {
    var groups = {};

    sites.forEach(function (site) {
        var sectionId = Number(site.sectionId || 0);
        var key = String(sectionId);

        if (!groups[key]) {
            groups[key] = {
                sectionId: sectionId,
                sectionName: getSectionName(sectionId),
                sites: []
            };
        }

        groups[key].sites.push(site);
    });

    var result = Object.keys(groups).map(function (key) {
        return groups[key];
    });

    result.sort(function (a, b) {
        if (a.sectionId === 0 && b.sectionId !== 0) return 1;
        if (a.sectionId !== 0 && b.sectionId === 0) return -1;

        var sectionA = allSections.find(function (s) { return Number(s.id || 0) === a.sectionId; });
        var sectionB = allSections.find(function (s) { return Number(s.id || 0) === b.sectionId; });

        var sortA = sectionA ? Number(sectionA.sort || 500) : 999999;
        var sortB = sectionB ? Number(sectionB.sort || 500) : 999999;

        if (sortA !== sortB) return sortA - sortB;

        return String(a.sectionName).localeCompare(String(b.sectionName));
    });

    return result;
}
```

---

## 7. Замени `renderSitesTable`

Полностью замени функцию `renderSitesTable(sites)` на эту:

```js
function renderSitesTable(sites) {
    var colspan = IS_BITRIX_ADMIN ? 10 : 2;

    if (!Array.isArray(sites) || !sites.length) {
        sitesTableBody.innerHTML = IS_BITRIX_ADMIN
            ? '<tr><td colspan="' + colspan + '">Сайтов пока нет</td></tr>'
            : '<div class="sb-user-sites-empty">Сайтов пока нет</div>';

        sitesCountBadge.textContent = '0 сайтов';
        return;
    }

    sitesCountBadge.textContent = sites.length + ' сайтов';

    var html = '';

    if (!IS_BITRIX_ADMIN) {
        var groups = groupSitesBySection(sites);

        groups.forEach(function (group) {
            html += ''
                + '<div class="sb-section-group">'
                + '  <h3 class="sb-section-title">' + escapeHtml(group.sectionName) + '</h3>'
                + '  <div class="sb-section-list">';

            group.sites.forEach(function (s) {
                var siteId = Number(s.id || 0);
                var currentUserRole = String(s.currentUserRole || 'VIEWER');
                var currentUserRoleRank = roleRank(currentUserRole);

                html += ''
                    + '<div class="sb-user-site-card">'
                    + '  <a class="sb-user-site-link" href="' + BASE_PATH + '/public.php?siteId=' + siteId + '">'
                    +        escapeHtml(s.name || '')
                    + '  </a>'
                    + '  <div class="sb-user-site-actions">'
                    +        (currentUserRoleRank >= 2
                                ? '<a class="sb-btn sb-btn-primary sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + siteId + '">Редактор</a>'
                                : '')
                    + '  </div>'
                    + '</div>';
            });

            html += ''
                + '  </div>'
                + '</div>';
        });

        sitesTableBody.innerHTML = html;
        return;
    }

    for (var i = 0; i < sites.length; i++) {
        var s = sites[i];
        var siteId = Number(s.id || 0);
        var status = siteStatus(s);
        var statusClass = status === 'published' ? 'success' : 'warning';
        var currentUserRole = String(s.currentUserRole || 'VIEWER');
        var currentUserRoleRank = roleRank(currentUserRole);

        html += ''
            + '<tr>'
            + '  <td>' + siteId + '</td>'
            + '  <td>'
            + '    <div class="sb-site-cell">'
            + '      <div class="sb-site-cell__title">' + escapeHtml(s.name || '') + '</div>'
            + '      <div class="sb-site-cell__meta">created: ' + escapeHtml(s.createdAt || '—') + '</div>'
            + '    </div>'
            + '  </td>'
            + '  <td>' + renderSectionSelect(s) + '</td>'
            + '  <td>' + roleBadge(currentUserRole) + '</td>'
            + '  <td>' + escapeHtml(s.slug || '') + '</td>'
            + '  <td><span class="sb-status-badge ' + statusClass + '">' + escapeHtml(status) + '</span></td>'
            + '  <td>' + renderGroupCell(s, currentUserRoleRank) + '</td>'
            + '  <td>' + yesNoBadge(Number(s.diskFolderId || 0) > 0) + '</td>'
            + '  <td>' + escapeHtml(s.updatedAt || '—') + '</td>'
            + '  <td>' + renderActions(s, currentUserRoleRank) + '</td>'
            + '</tr>';
    }

    sitesTableBody.innerHTML = html;
}
```

---

## 8. Замени `loadSites`

Было:

```js
function loadSites() {
    api('site.list', {}, function (res) {
        ...
    });
}
```

Сделай так:

```js
function loadSites() {
    loadSections(function () {
        api('site.list', {}, function (res) {
            if (!res || res.ok !== true) {
                sitesTableBody.innerHTML = IS_BITRIX_ADMIN
                    ? '<tr><td colspan="10">Не удалось загрузить данные</td></tr>'
                    : '<div class="sb-user-sites-empty">Не удалось загрузить данные</div>';

                sitesCountBadge.textContent = 'Ошибка загрузки';
                return;
            }

            allSites = Array.isArray(res.sites) ? res.sites : [];
            applySearch();
        });
    });
}
```

---

## 9. Добавь создание секции и назначение сайта в секцию

Добавь функции:

```js
function openSectionModal() {
    if (!IS_BITRIX_ADMIN) return;

    var modal = document.getElementById('createSectionModal');
    if (!modal) return;

    modal.hidden = false;

    setTimeout(function () {
        var input = document.getElementById('sectionName');
        if (input) input.focus();
    }, 50);
}

function closeSectionModal() {
    var modal = document.getElementById('createSectionModal');
    if (!modal) return;

    modal.hidden = true;
}

function createSection() {
    var nameInput = document.getElementById('sectionName');
    var sortInput = document.getElementById('sectionSort');

    if (!nameInput) return;

    var name = String(nameInput.value || '').trim();
    var sort = Number(sortInput && sortInput.value ? sortInput.value : 500);

    if (!name) {
        alert('Введите название секции');
        nameInput.focus();
        return;
    }

    api('section.create', {
        name: name,
        sort: sort
    }, function () {
        nameInput.value = '';
        if (sortInput) sortInput.value = '500';

        closeSectionModal();
        loadSites();
    }, function (err) {
        var message = err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR';
        alert('Не удалось создать секцию: ' + message);
    });
}

function setSiteSection(siteId, sectionId) {
    api('site.setSection', {
        siteId: siteId,
        sectionId: sectionId
    }, function () {
        loadSites();
    }, function (err) {
        var message = err && (err.error || err.message) ? (err.error || err.message) : 'UNKNOWN_ERROR';
        alert('Не удалось назначить секцию: ' + message);
        loadSites();
    });
}
```

---

## 10. Обработчики событий

Найди обработчик:

```js
sitesTableBody.addEventListener('click', function (e) {
```

После него добавь ещё один обработчик `change`:

```js
sitesTableBody.addEventListener('change', function (e) {
    var select = e.target.closest('[data-action="set-section"]');
    if (!select) return;

    var siteId = Number(select.getAttribute('data-site-id') || 0);
    var sectionId = Number(select.value || 0);

    if (siteId <= 0) return;

    setSiteSection(siteId, sectionId);
});
```

И внизу, рядом с кнопками создания сайта, добавь:

```js
var createSectionBtn = document.getElementById('createSectionBtn');
if (createSectionBtn) {
    createSectionBtn.addEventListener('click', openSectionModal);
}

var saveSectionBtn = document.getElementById('saveSectionBtn');
if (saveSectionBtn) {
    saveSectionBtn.addEventListener('click', createSection);
}

document.querySelectorAll('[data-close-section-modal]').forEach(function (btn) {
    btn.addEventListener('click', closeSectionModal);
});
```

А в обработчик `Escape` добавь закрытие секционной модалки:

```js
document.addEventListener('keydown', function (e) {
    if (e.key === 'Escape') {
        closeCreateSiteModal();
        closeSectionModal();
    }
});
```

---

После этого админ сможет создавать секции и назначать сайты в секции, а обычный пользователь увидит уже сгруппированный компактный список.
