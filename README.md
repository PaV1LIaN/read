Отлично. Делаем **по-взрослому и безопасно**: **удаление меню — только OWNER**, плюс защита `topMenuId` (если меню удалили/не существует).

Ниже даю **готовые вставки**: что добавить в `api.php` и что поправить/добавить в `menu.php`.

---

# 1) api.php — добавить `site.get`, `menu.setTop`, `menu.delete` + авто-сброс topMenuId

### 1.1. Добавь хелпер “найти сайт”

В `api.php` рядом с другими helper-функциями (например после `sb_site_exists()`), вставь:

```php
function sb_find_site(int $siteId): ?array {
    foreach (sb_read_sites() as $s) {
        if ((int)($s['id'] ?? 0) === $siteId) return $s;
    }
    return null;
}
```

---

### 1.2. Добавь action `site.get`

Поставь **после** `site.list` (или в блоке SITES, до `site.create` — не важно, но логично там).

```php
if ($action === 'site.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_viewer($siteId);

    $site = sb_find_site($siteId);
    if (!$site) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    echo json_encode(['ok'=>true,'site'=>$site], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

### 1.3. Добавь action `menu.setTop`

Если у тебя его ещё нет в `api.php` — вставь **в секции MENU** (рядом с `menu.create`/`menu.update`):

```php
if ($action === 'menu.setTop') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    // безопаснее для серьёзного проекта: только OWNER
    sb_require_owner($siteId);

    // проверим что меню существует у этого siteId
    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    $menus = $rec ? ($rec['menus'] ?? []) : [];
    $exists = false;
    foreach ($menus as $m) {
        if ((int)($m['id'] ?? 0) === $menuId) { $exists = true; break; }
    }
    if (!$exists) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $sites = sb_read_sites();
    $found = false;
    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $s['topMenuId'] = $menuId;
            $s['updatedAt'] = date('c');
            $s['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($s);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_sites($sites);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

### 1.4. Добавь action `menu.delete` (и сброс topMenuId если удалили верхнее)

В секции MENU вставь:

```php
if ($action === 'menu.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    // серьёзно и безопасно: удаление меню только OWNER
    sb_require_owner($siteId);

    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $menus = $rec['menus'] ?? [];
    $before = count($menus);
    $menus = array_values(array_filter($menus, fn($m) => (int)($m['id'] ?? 0) !== $menuId));
    if (count($menus) === $before) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $rec['menus'] = $menus;
    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    // если удалили верхнее меню — сбросим topMenuId
    $sites = sb_read_sites();
    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            if ((int)($s['topMenuId'] ?? 0) === $menuId) {
                $s['topMenuId'] = 0;
                $s['updatedAt'] = date('c');
                $s['updatedBy'] = (int)$USER->GetID();
            }
            break;
        }
    }
    unset($s);
    sb_write_sites($sites);

    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

✅ На этом API готов: можно удалять меню и не ломать `view.php`.

---

# 2) menu.php — исправить loadAll + кнопка “Удалить меню” + авто “верхнее” при первом меню

Ты прислал `menu.php`, там есть 2 проблемы:

1. `loadAll()` — у тебя `api('site.get')`, но переменная в return `siteRes`/`siteRes.site` перепутана (`siteRes` не существует).
2. Нет UI для удаления меню.

### 2.1. Исправь loadAll()

Замени свой `loadAll()` на этот:

```js
async function loadAll() {
  const [menusRes, pagesRes, siteRes] = await Promise.all([
    api('menu.list'),
    api('page.list'),
    api('site.get', { siteId })
  ]);

  if (!menusRes || menusRes.ok !== true) throw new Error('menu.list failed');
  if (!pagesRes || pagesRes.ok !== true) throw new Error('page.list failed');
  if (!siteRes || siteRes.ok !== true) throw new Error('site.get failed');

  const site = siteRes.site || {};
  return { menus: menusRes.menus || [], pages: pagesRes.pages || [], topMenuId: parseInt(site.topMenuId || 0, 10) || 0 };
}
```

> Обрати внимание: `api('site.get', { siteId })` — чтобы `siteId` точно ушёл.

---

### 2.2. В render() добавь кнопку “Удалить меню”

В твоём `render({menus, pages, topMenuId})` найди блок кнопок:

```js
<div class="btns">
    ...
    <button class="ui-btn ui-btn-light ui-btn-xs" data-menu-rename="${m.id}">Переименовать</button>
    <button class="ui-btn ui-btn-primary ui-btn-xs" data-item-add="${m.id}">+ Пункт</button>
</div>
```

И допиши между “Переименовать” и “+ Пункт” (или как хочешь):

```js
<button class="ui-btn ui-btn-danger ui-btn-xs" data-menu-delete="${m.id}">Удалить меню</button>
```

---

### 2.3. В обработчик кликов добавь `data-menu-delete`

Внизу, внутри `document.addEventListener('click', ...)` добавь блок (после rename — ок):

```js
const md = e.target.closest('[data-menu-delete]');
if (md) {
  const menuId = parseInt(md.getAttribute('data-menu-delete'), 10);

  BX.UI.Dialogs.MessageBox.show({
    title: 'Удалить меню #' + menuId + '?',
    message: 'Удалить меню целиком и все его пункты? (только OWNER)',
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      api('menu.delete', { menuId }).then(res => {
        if (!res || res.ok !== true) {
          BX.UI.Notification.Center.notify({ content: 'Не удалось удалить меню (нужен OWNER)' });
          return;
        }
        BX.UI.Notification.Center.notify({ content: 'Меню удалено' });
        mb.close();
        refresh();
      }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка menu.delete' }));
    }
  });

  return;
}
```

---

### 2.4. (Рекомендую) Авто назначать topMenuId при создании первого меню

В `createMenu()` внутри `api('menu.create'...)` — после успешного создания:

Найди:

```js
BX.UI.Notification.Center.notify({ content: 'Меню создано' });
mb.close();
refresh();
```

Замени на:

```js
BX.UI.Notification.Center.notify({ content: 'Меню создано' });
mb.close();

// если верхнее меню ещё не назначено — назначим автоматически (удобно и логично)
api('site.get', { siteId }).then(sres => {
  const top = (sres && sres.ok === true) ? (parseInt(sres.site?.topMenuId || 0, 10) || 0) : 0;

  if (!top) {
    // menuId пришёл в res.menu.id
    const newMenuId = parseInt(res.menu?.id || 0, 10);
    if (newMenuId) {
      api('menu.setTop', { menuId: newMenuId }).finally(() => refresh());
      return;
    }
  }
  refresh();
});
```

---

# 3) Важный момент по правам

Я сделал так:

* `menu.setTop` → **OWNER**
* `menu.delete` → **OWNER**

Это именно “серьёзный проект” вариант: никто случайно не сломает навигацию.

---

Если ты это вставил — следующий шаг (без роутинга) я бы делал такой:
**“Главная страница” в index.php (UI кнопка) + индикатор**, потому что у тебя уже есть `site.setHome` в API, просто нужно довести интерфейс.
