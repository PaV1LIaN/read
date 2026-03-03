В целом ты вставил **почти правильно**, но есть **2 критичные ошибки**, из-за которых PHP либо упадёт, либо будет warning/notice и меню может не работать как ожидаешь.

---

## ✅ 1) Критичная ошибка: `$topMenuId` берёшь ДО проверки `$site`

Сейчас у тебя:

```php
$site = null;
foreach (...) { ... }

$topMenuId = (int)($site['topMenuId'] ?? 0);

$page = null;
...
if (!$site || !$page) { ... }
```

Если сайт **не найден**, то `$site = null`, и выражение `$site['topMenuId']` вызовет **Fatal error** в PHP 8+: *Trying to access array offset on value of type null*.

✅ Правильно так:

```php
$site = null;
foreach ($sites as $s) {
    if ((int)($s['id'] ?? 0) === $siteId) { $site = $s; break; }
}

$page = null;
foreach ($pages as $p) {
    if ((int)($p['id'] ?? 0) === $pageId && (int)($p['siteId'] ?? 0) === $siteId) { $page = $p; break; }
}

if (!$site || !$page) { ... exit; }

$topMenuId = (int)($site['topMenuId'] ?? 0);
```

То есть **`$topMenuId` вычисляем только после проверки**, что `$site` найден.

---

## ✅ 2) Критичная ошибка: ты оставил кусок с `$selectedMenu`, которого нет

У тебя после цикла выбора меню стоит:

```php
if ($selectedMenu) {
    $menuItems = $selectedMenu['items'] ?? [];
    ...
}
```

Но переменной `$selectedMenu` **вообще нет**, у тебя она нигде не объявлена — будет Notice/Warning, а в строгих настройках может ломать вывод.

✅ Просто **удали этот блок полностью**:

```php
if ($selectedMenu) { ... }
```

Он лишний — у тебя уже есть `$picked`, и ты уже наполнил `$menuItems`.

---

## ✅ Всё остальное по меню (`topMenuId`) — нормально

Этот блок у тебя ок:

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

## Мини-патч (что именно сделать)

1. Перенеси строку `$topMenuId = ...` **ниже** `if (!$site || !$page) exit;`
2. Удали блок `if ($selectedMenu) { ... }`

---

Если хочешь — следующим шагом я предложу “правильный” UX для вложенности на `view.php`:

* хлебные крошки уже есть ✅
* добавим “Назад в раздел” + “Следующая/Предыдущая” среди соседей в разделе (очень удобно).
