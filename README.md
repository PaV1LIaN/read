Да, можно. Я бы сделал так:

1. Для обычного пользователя главная страница — **не таблица**, а компактный список сайтов.
2. Название сайта — кликабельное, открывает публичную страницу.
3. Кнопка **«Редактор»** показывается только если роль `EDITOR / ADMIN / OWNER`.
4. Секции сайтов добавляем отдельной сущностью: например **«Отделы», «Проекты», «Информационные порталы», «Сервисы»**.

Сначала быстро поправим внешний вид обычного пользователя.

## 1. В `index.php` добавь CSS

В `<style>` добавь:

```css
.sb-user-site-list {
    display: flex;
    flex-direction: column;
    gap: 8px;
}

.sb-user-site-card {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 14px;
    padding: 12px 14px;
    border: 1px solid #eef2f7;
    border-radius: 14px;
    background: #fff;
}

.sb-user-site-card:hover {
    border-color: #c7d2fe;
    background: #fcfcff;
}

.sb-user-site-link {
    color: #111827;
    font-size: 15px;
    font-weight: 700;
    text-decoration: none;
}

.sb-user-site-link:hover {
    color: #2563eb;
    text-decoration: underline;
}

.sb-user-site-actions {
    display: flex;
    gap: 8px;
    flex-shrink: 0;
}

.sb-user-sites-empty {
    padding: 18px;
    color: #6b7280;
}
```

---

## 2. Замени HTML блока списка сайтов

Найди блок:

```php
<div class="sb-dash-table-wrap">
    <table class="sb-dash-table <?= $isBitrixAdmin ? '' : 'sb-user-sites-table' ?>">
        ...
    </table>
</div>
```

Замени на:

```php
<?php if ($isBitrixAdmin): ?>
    <div class="sb-dash-table-wrap">
        <table class="sb-dash-table">
            <thead>
            <tr>
                <th>ID</th>
                <th>Сайт</th>
                <th>Роль</th>
                <th>Slug</th>
                <th>Домашняя</th>
                <th>Группа Битрикс24</th>
                <th>Диск</th>
                <th>Изменен</th>
                <th>Действия</th>
            </tr>
            </thead>

            <tbody id="sitesTableBody">
            <tr>
                <td colspan="9">Загрузка...</td>
            </tr>
            </tbody>
        </table>
    </div>
<?php else: ?>
    <div id="sitesTableBody" class="sb-user-site-list">
        <div class="sb-user-sites-empty">Загрузка...</div>
    </div>
<?php endif; ?>
```

---

## 3. Замени часть обычного пользователя в `renderSitesTable`

В функции `renderSitesTable(sites)` найди этот кусок:

```js
if (!IS_BITRIX_ADMIN) {
    html += ''
        + '<tr>'
        + '  <td>'
        + '    <div class="sb-site-cell">'
        + '      <div class="sb-site-cell__title">' + escapeHtml(s.name || '') + '</div>'
        + '    </div>'
        + '  </td>'
        + '  <td style="text-align:right;">' + renderUserActions(s, currentUserRoleRank) + '</td>'
        + '</tr>';

    continue;
}
```

Замени на:

```js
if (!IS_BITRIX_ADMIN) {
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

    continue;
}
```

И в начале функции замени пустое состояние:

```js
sitesTableBody.innerHTML = '<tr><td colspan="' + colspan + '">Сайтов пока нет</td></tr>';
```

на:

```js
sitesTableBody.innerHTML = IS_BITRIX_ADMIN
    ? '<tr><td colspan="' + colspan + '">Сайтов пока нет</td></tr>'
    : '<div class="sb-user-sites-empty">Сайтов пока нет</div>';
```

Так у обычного пользователя получится компактно:

```text
321                                     
Информационный портал "Сервис Строка"
```

Клик по названию откроет сайт.

---

# Про секции сайтов

Да, это правильная идея. Я бы сделал отдельную таблицу:

```sql
CREATE TABLE IF NOT EXISTS sitebuilder.site_section (
    id bigserial PRIMARY KEY,
    name varchar(255) NOT NULL,
    sort int NOT NULL DEFAULT 500,
    created_by int,
    created_at timestamp without time zone DEFAULT now(),
    updated_by int,
    updated_at timestamp without time zone DEFAULT now()
);

ALTER TABLE sitebuilder.site
ADD COLUMN IF NOT EXISTS section_id bigint NULL;
```

Дальше логика такая:

```text
Секция
 ├── Сайт 1
 ├── Сайт 2
 └── Сайт 3
```

Для администратора Битрикс24:

```text
[Создать секцию]
[Переименовать секцию]
[Удалить секцию]
[Назначить сайт в секцию]
```

Для обычного пользователя:

```text
Информационные порталы
  - 321
  - Сервис Строка

Внутренние сервисы
  - Сайт отдела кадров
  - База знаний
```

Следующим шагом лучше сделать именно **структуру секций**: таблицу, `storage_db.php`, API `section.list/create/update/delete`, потом группировку сайтов на главной.
