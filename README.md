Отлично, идём дальше по “серьёзному” пути: **фиксируем данные и делаем `topMenuId` реальным источником правды.**
Теперь нужно **2 action’а в `api.php`** + **1 правка в `view.php`**, чтобы меню отображалось именно то, которое назначено “верхним”.

---

# A) api.php — добавляем `site.get`

**Куда вставлять:** в `api.php` в блоке `/** -------------------- SITES -------------------- */` **после** `site.list` (или перед ним — не принципиально, главное до `http_response_code(400)`).

Вставь:

```php
if ($action === 'site.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    if ($siteId <= 0) {
        http_response_code(422);
        echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE);
        exit;
    }

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

---

# B) api.php — добавляем `menu.setTop`

**Куда вставлять:** в `api.php` в блоке `/** -------------------- MENU -------------------- */`
Лучше всего **после** `menu.update` (rename) и **до** `menu.item.add`.

Вставь:

```php
if ($action === 'menu.setTop') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    // проверим, что такое меню реально существует у этого сайта
    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $found = false;
    foreach (($rec['menus'] ?? []) as $m) {
        if ((int)($m['id'] ?? 0) === $menuId) { $found = true; break; }
    }
    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    // пишем topMenuId в sites.json
    $sites = sb_read_sites();
    $ok = false;
    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $s['topMenuId'] = $menuId;
            $s['updatedAt'] = date('c');
            $s['updatedBy'] = (int)$USER->GetID();
            $ok = true;
            break;
        }
    }
    unset($s);

    if (!$ok) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'SITE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_sites($sites);

    echo json_encode(['ok'=>true,'topMenuId'=>$menuId], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

# C) view.php — показываем именно верхнее меню (`topMenuId`)

Сейчас `view.php` берёт **первое меню**. Это неправильно для серьёзного проекта.

### 1) Считаем `topMenuId` у сайта

После того как нашли `$site` (у тебя блок “find site/page”), добавь:

```php
$topMenuId = (int)($site['topMenuId'] ?? 0);
```

### 2) Заменяем блок “menu: take first menu”

Найди в `view.php`:

```php
// menu: take first menu for this site (if exists)
$menuItems = [];
foreach ($menusAll as $rec) {
    if ((int)($rec['siteId'] ?? 0) === $siteId) {
        $menus = $rec['menus'] ?? [];
        if (is_array($menus) && count($menus) > 0) {
            $first = $menus[0];
            $menuItems = $first['items'] ?? [];
            if (!is_array($menuItems)) $menuItems = [];
            usort($menuItems, fn($a, $b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
        }
        break;
    }
}
```

И **замени полностью** на:

```php
// menu: take topMenuId for this site (if set), otherwise fallback to first menu
$menuItems = [];
foreach ($menusAll as $rec) {
    if ((int)($rec['siteId'] ?? 0) !== $siteId) continue;

    $menus = $rec['menus'] ?? [];
    if (!is_array($menus) || !count($menus)) break;

    $picked = null;

    if ($topMenuId > 0) {
        foreach ($menus as $m) {
            if ((int)($m['id'] ?? 0) === $topMenuId) { $picked = $m; break; }
        }
    }

    if (!$picked) $picked = $menus[0];

    $menuItems = $picked['items'] ?? [];
    if (!is_array($menuItems)) $menuItems = [];
    usort($menuItems, fn($a, $b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
    break;
}
```

---

# D) Что будет в итоге (логика “как должно работать”)

* В `menu.php` ты назначаешь конкретное меню как верхнее → `sites.json.topMenuId = X`
* `view.php` при показе страницы берёт **именно это меню X**
* Если `topMenuId` не задан — работает fallback на первое меню

---

Если ты уже добавил кнопки в `menu.php` и всё ок — **следующий серьёзный шаг**:
**в index.php рядом с “Меню” показывать, какое меню назначено верхним (и предупреждать, если не назначено).**
Хочешь — сразу дам готовый патч для `index.php` (минимальный, без “переписывания дизайна”).
