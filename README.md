Отлично, `api.php` как раз то что нужно. Делаем “правильнее”: **`sites.json` хранит `topMenuId`**, и `view.php` показывает именно его (с fallback).

Ниже даю **готовые вставки/патчи** для трёх файлов: `api.php`, `menu.php`, `view.php`.
(Старался минимально вмешиваться в твой код.)

---

# 1) Правка `api.php`

## 1.1. В `site.create` добавь поле `topMenuId`

Найди блок:

```php
$site = [
  'id' => $id,
  'name' => $name,
  'slug' => $slug,
  'createdBy' => (int)$USER->GetID(),
  'createdAt' => date('c'),
  'diskFolderId' => 0,
];
```

Замени на:

```php
$site = [
    'id' => $id,
    'name' => $name,
    'slug' => $slug,
    'createdBy' => (int)$USER->GetID(),
    'createdAt' => date('c'),
    'diskFolderId' => 0,

    // NEW: какое меню считать верхним (0 = не задано)
    'topMenuId' => 0,
];
```

---

## 1.2. Автовыбор “верхнего меню” при первом `menu.create` (не обязательно, но удобно)

В `menu.create` после того как ты сохранил `menus.json`:

Сейчас у тебя конец `menu.create` такой:

```php
$rec['menus'][] = $menu;
sb_menu_upsert_site_record($all, $siteId, $rec);
sb_write_menus($all);

echo json_encode(['ok' => true, 'menu' => $menu], JSON_UNESCAPED_UNICODE);
exit;
```

Сделай так (добавим авто-проставление topMenuId, если ещё 0):

```php
$rec['menus'][] = $menu;
sb_menu_upsert_site_record($all, $siteId, $rec);
sb_write_menus($all);

// NEW: если topMenuId еще не задан — сделать первое меню верхним
$sites = sb_read_sites();
$changedSite = false;
foreach ($sites as &$s) {
    if ((int)($s['id'] ?? 0) === $siteId) {
        $top = (int)($s['topMenuId'] ?? 0);
        if ($top <= 0) {
            $s['topMenuId'] = (int)$menuId;
            $s['updatedAt'] = date('c');
            $s['updatedBy'] = (int)$USER->GetID();
            $changedSite = true;
        }
        break;
    }
}
unset($s);

if ($changedSite) sb_write_sites($sites);

echo json_encode(['ok' => true, 'menu' => $menu], JSON_UNESCAPED_UNICODE);
exit;
```

---

## 1.3. Добавь новый action: `menu.setTop` (EDITOR+)

Вставь этот блок **после `menu.update`** (или вообще внутри секции MENU — главное до финального UNKNOWN_ACTION):

```php
// menu.setTop (EDITOR+): назначить верхнее меню сайта
if ($action === 'menu.setTop') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    // убедимся, что меню существует у этого сайта
    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $menu = sb_menu_find_menu($rec, $menuId);
    if (!$menu) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    // пишем в sites.json
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

✅ Всё. API готов.

---

# 2) Правка `view.php` — брать меню по `topMenuId`

Сейчас у тебя в `view.php` логика такая: “берём первое меню”.

Нужно заменить на:

* читаем `topMenuId` из `$site`
* в `menus.json` ищем меню с этим id
* если нет — fallback на первое меню как раньше

## 2.1. Найди блок:

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

## 2.2. Замени на:

```php
// menu: берем topMenuId из sites.json (fallback на первое меню)
$menuItems = [];
$topMenuId = (int)($site['topMenuId'] ?? 0);

foreach ($menusAll as $rec) {
    if ((int)($rec['siteId'] ?? 0) !== $siteId) continue;

    $menus = $rec['menus'] ?? [];
    if (!is_array($menus) || !count($menus)) break;

    $picked = null;

    // 1) пробуем найти по topMenuId
    if ($topMenuId > 0) {
        foreach ($menus as $m) {
            if ((int)($m['id'] ?? 0) === $topMenuId) { $picked = $m; break; }
        }
    }

    // 2) fallback на первое
    if (!$picked) $picked = $menus[0];

    $menuItems = $picked['items'] ?? [];
    if (!is_array($menuItems)) $menuItems = [];
    usort($menuItems, fn($a, $b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));
    break;
}
```

---

# 3) Правка `menu.php` — кнопка “Сделать верхним”

## 3.1. Перед `function render({menus, pages})` добавь переменную `topMenuId`

Тебе нужно, чтобы `menu.php` знал текущий `topMenuId`.
Проще всего: добавить API `site.get`… но у тебя его нет. Поэтому сделаем **лёгкий вариант**:

* просто добавим в `menu.list` в `api.php` поле `topMenuId` (в ответе)
* и `menu.php` будет показывать кнопку и подсветку

### 3.1.1. Добавь в `api.php` в `menu.list` возврат topMenuId

Найди `menu.list`:

```php
echo json_encode(['ok' => true, 'menus' => $menus], JSON_UNESCAPED_UNICODE);
exit;
```

Замени на:

```php
$sites = sb_read_sites();
$topMenuId = 0;
foreach ($sites as $s) {
    if ((int)($s['id'] ?? 0) === $siteId) { $topMenuId = (int)($s['topMenuId'] ?? 0); break; }
}

echo json_encode(['ok' => true, 'menus' => $menus, 'topMenuId' => $topMenuId], JSON_UNESCAPED_UNICODE);
exit;
```

---

## 3.2. В `menu.php` обнови `loadAll()` чтобы возвращал `topMenuId`

Найди:

```js
async function loadAll() {
  const [menusRes, pagesRes] = await Promise.all([
    api('menu.list'),
    api('page.list')
  ]);
  ...
  return { menus: menusRes.menus || [], pages: pagesRes.pages || [] };
}
```

Замени `return` на:

```js
return {
  menus: menusRes.menus || [],
  pages: pagesRes.pages || [],
  topMenuId: parseInt(menusRes.topMenuId || 0, 10) || 0
};
```

---

## 3.3. В `render({menus, pages})` добавь `topMenuId` и кнопку

Найди сигнатуру:

```js
function render({menus, pages}) {
```

Замени на:

```js
function render({menus, pages, topMenuId}) {
```

И внутри `menus.map(m => { ... })` в кнопки добавь:

### было:

```js
<div class="btns">
  <button class="ui-btn ui-btn-light ui-btn-xs" data-menu-rename="${m.id}">Переименовать</button>
  <button class="ui-btn ui-btn-primary ui-btn-xs" data-item-add="${m.id}">+ Пункт</button>
</div>
```

### сделай:

```js
<div class="btns">
  ${parseInt(m.id,10) === (topMenuId||0)
    ? `<span class="muted small" style="padding:6px 8px;">Верхнее меню</span>`
    : `<button class="ui-btn ui-btn-success ui-btn-xs" data-menu-set-top="${m.id}">Сделать верхним</button>`
  }
  <button class="ui-btn ui-btn-light ui-btn-xs" data-menu-rename="${m.id}">Переименовать</button>
  <button class="ui-btn ui-btn-primary ui-btn-xs" data-item-add="${m.id}">+ Пункт</button>
</div>
```

(Если `ui-btn-success` у тебя не нравится — можно `ui-btn-primary`.)

---

## 3.4. В обработчик кликов `document.addEventListener('click'...)` добавь обработку setTop

В конце обработчика (рядом с rename/add/del/move) вставь:

```js
const st = e.target.closest('[data-menu-set-top]');
if (st) {
  const menuId = parseInt(st.getAttribute('data-menu-set-top'), 10);
  api('menu.setTop', { menuId }).then(res => {
    if (!res || res.ok !== true) {
      BX.UI.Notification.Center.notify({ content: 'Не удалось назначить верхнее меню (нужен EDITOR+)' });
      return;
    }
    BX.UI.Notification.Center.notify({ content: 'Верхнее меню назначено' });
    refresh();
  }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка menu.setTop' }));
  return;
}
```

---

# Итого что получится

* При создании сайта: `topMenuId=0`
* При создании первого меню: оно автоматически станет верхним (если ты вставил пункт 1.2)
* В `menu.php` можно руками назначать “верхнее”
* В `view.php` будет показываться именно оно

---

## Дальше

Если хочешь — следующим шагом сделаем “красиво”:

* выбор в `menu.php`: **где показывать меню** (top/footer/side)
* возможность в `view.php` рендерить разные меню по позициям

Но сначала проверь, что всё работает по патчу выше.

Если после вставок что-то упадёт — просто пришли ошибку из консоли/ответ API, я точечно поправлю.
