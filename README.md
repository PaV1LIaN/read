([Past chat][1])([Past chat][1])([Past chat][1])([Past chat][1])

Ок, тогда делаем “как для серьёзного проекта”: **иерархия страниц = структура (sitemap/хлебные крошки/URL-логика), а верхнее меню выбирается явно у сайта**.

Сейчас у тебя меню в `view.php` берётся “первое попавшееся” — это и есть причина, почему непонятно где оно должно отображаться и что выбирать. Делаем правильно: **в `sites.json` добавляем `topMenuId`**, и в UI (menu.php) появится кнопка “Сделать верхним”.

Ниже — конкретные правки.

---

## 1) Миграция данных (один раз)

В `sites.json` у каждого сайта появится поле:

```json
"topMenuId": 0
```

Пока можешь руками не править — мы сделаем “автозаполнение по умолчанию” при `menu.create` и “ленивую миграцию” в `view.php`.

---

## 2) api.php — автопроставление topMenuId при первом меню

Найди блок `if ($action === 'menu.create') { ... }` и **после сохранения меню** добавь:

```php
// если это первое меню у сайта — сделаем его верхним (topMenuId)
try {
    $sites = sb_read_sites();
    $changed = false;
    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $curTop = (int)($s['topMenuId'] ?? 0);
            if ($curTop <= 0) {
                $s['topMenuId'] = (int)$menuId;
                $s['updatedAt'] = date('c');
                $s['updatedBy'] = (int)$USER->GetID();
                $changed = true;
            }
            break;
        }
    }
    unset($s);
    if ($changed) sb_write_sites($sites);
} catch (\Throwable $e) {
    // молча, это не критично
}
```

Так у нового сайта первое созданное меню автоматически становится верхним.

---

## 3) api.php — новый action: site.setTopMenu

В конец файла **перед** `UNKNOWN_ACTION` добавь новый обработчик:

```php
if ($action === 'site.setTopMenu') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    // проверим, что меню реально принадлежит сайту
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

## 4) view.php — выбирать меню по topMenuId (и “ленивая миграция”)

Сейчас у тебя: “берём первое меню”. Заменяем логику на:

* читаем `topMenuId` из `$site`
* если его нет/0 — берём первое меню и **(опционально)** запоминаем в `sites.json` (лениво)

В `view.php` найди блок:

```php
// menu: take first menu for this site (if exists)
$menuItems = [];
foreach ($menusAll as $rec) {
 ...
}
```

И замени на:

```php
$menuItems = [];
$topMenuId = (int)($site['topMenuId'] ?? 0);

$selectedMenu = null;
$siteMenusRecIndex = null;

foreach ($menusAll as $i => $rec) {
    if ((int)($rec['siteId'] ?? 0) !== $siteId) continue;
    $siteMenusRecIndex = $i;

    $menus = $rec['menus'] ?? [];
    if (!is_array($menus) || !count($menus)) break;

    // 1) пробуем найти по topMenuId
    if ($topMenuId > 0) {
        foreach ($menus as $m) {
            if ((int)($m['id'] ?? 0) === $topMenuId) { $selectedMenu = $m; break; }
        }
    }

    // 2) если topMenuId пустой/битый — берём первый
    if (!$selectedMenu) {
        $selectedMenu = $menus[0];

        // ленивая миграция: запомним выбранное меню как верхнее
        if ($topMenuId <= 0 && isset($selectedMenu['id'])) {
            try {
                $sitesAll = sb_read_json('sites.json'); // у тебя уже есть sb_read_json выше
                foreach ($sitesAll as &$ss) {
                    if ((int)($ss['id'] ?? 0) === $siteId) {
                        $ss['topMenuId'] = (int)$selectedMenu['id'];
                        break;
                    }
                }
                unset($ss);
                // и запишем обратно
                file_put_contents(sb_data_path('sites.json'), json_encode(array_values($sitesAll), JSON_UNESCAPED_UNICODE | JSON_PRETTY_PRINT));
            } catch (\Throwable $e) {}
        }
    }
    break;
}

if ($selectedMenu) {
    $menuItems = $selectedMenu['items'] ?? [];
    if (!is_array($menuItems)) $menuItems = [];
    usort($menuItems, fn($a,$b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
}
```

> Да, тут “ленивая запись” через `file_put_contents` — чтобы не тащить все функции записи из api.php в view.php. Если хочешь “совсем правильно”, вынесем storage-функции в общий `lib/storage.php` и подключим.

---

## 5) menu.php — кнопка “Сделать верхним” (UI)

Чтобы управлять этим из интерфейса, `menu.php` должен знать `topMenuId` сайта. Самый простой путь без новых API: **добавим action `site.list` уже возвращает сайт**, но menu.php сейчас не загружает site. Поэтому проще добавить новый action `site.get`.

### Быстро: добавим action site.get в api.php

В api.php добавь:

```php
if ($action === 'site.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    sb_require_viewer($siteId);

    $sites = sb_read_sites();
    foreach ($sites as $s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            echo json_encode(['ok'=>true,'site'=>$s], JSON_UNESCAPED_UNICODE);
            exit;
        }
    }
    http_response_code(404);
    echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
    exit;
}
```

### menu.php (JS)

1. В `loadAll()` добавь третий запрос `site.get`:

```js
const [menusRes, pagesRes, siteRes] = await Promise.all([
  api('menu.list'),
  api('page.list'),
  api('site.get')
]);
...
return { menus: menusRes.menus||[], pages: pagesRes.pages||[], site: siteRes.site||{} };
```

2. В `render({menus,pages,site})` получи `topMenuId`:

```js
const topMenuId = parseInt(site?.topMenuId || 0, 10) || 0;
```

3. В карточку меню добавь кнопку и бейдж:

```js
const isTop = parseInt(m.id,10) === topMenuId;
...
<div>
  <b>...</b>
  ${isTop ? '<span class="muted small" style="margin-left:8px; background:#eef2ff; padding:2px 8px; border-radius:999px; color:#1d4ed8;">TOP</span>' : ''}
</div>
...
<button class="ui-btn ui-btn-light ui-btn-xs" data-menu-top="${m.id}">Сделать верхним</button>
```

4. В обработчик кликов добавь:

```js
const top = e.target.closest('[data-menu-top]');
if (top) {
  const menuId = parseInt(top.getAttribute('data-menu-top'), 10);
  api('site.setTopMenu', { menuId }).then(r=>{
    if(!r || r.ok!==true){ BX.UI.Notification.Center.notify({content:'Не удалось (нужен EDITOR+)'}); return; }
    BX.UI.Notification.Center.notify({content:'Верхнее меню обновлено'});
    refresh();
  });
  return;
}
```

---

# Как должна работать вложенность (правильно)

Для “серьёзного проекта” вложенность страниц обычно используется так:

1. **Структура (Sitemap):** дерево страниц для админа/редактора.
2. **Навигация:**

   * хлебные крошки (родители → текущая)
   * “в этом разделе” (дети текущей)
3. **URL-логика:** либо `/parent/child`, либо хотя бы вычисляемый “path” для навигации/SEO (даже если реальный роутинг пока не нужен).
4. **Меню отдельно:** верхнее меню — это **выборка страниц/URL**, не обязано совпадать с деревом.

Сейчас мы как раз отделили **меню** от **дерева**. Следующий “правильный” шаг по дереву — сделать в `view.php`:

* хлебные крошки по `parentId`
* блок “дочерние страницы” (если есть дети)

Если ок — в следующем сообщении я дам готовый патч для `view.php`: *breadcrumbs + children navigation* (без rewrite и роутинга).

[1]: https://chatgpt.com/c/6997ed68-f66c-8395-8b8f-7fa8af7d70b9 "Создание сайта на Битрикс24"
