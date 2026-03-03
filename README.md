Ок, идём дальше. Раз у тебя **`site.get` уже есть**, следующий логичный “серьёзный” набор пунктов для меню — это:

1. **API: `menu.setTop`** (сохранять `topMenuId` в `sites.json`)
2. **API: `menu.delete`** (удаление меню целиком)
3. **Безопасность/целостность:** если удалили меню, которое было `topMenuId` → **сбросить `topMenuId` в 0**
4. **UI menu.php:** кнопка “Удалить меню” + подтверждение
5. **view.php:** ты уже сделал чтение `topMenuId` — супер (там останется как есть)

Ниже — готовые вставки.

---

## 1) api.php — добавить action `menu.setTop`

Вставь **в секцию MENU** (рядом с `menu.create/menu.update/...`), например после `menu.update`:

```php
// menu.setTop (EDITOR+): назначить меню верхним
if ($action === 'menu.setTop') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    // проверим, что такое меню вообще существует у сайта
    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $exists = false;
    foreach (($rec['menus'] ?? []) as $m) {
        if ((int)($m['id'] ?? 0) === $menuId) { $exists = true; break; }
    }
    if (!$exists) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    // записываем topMenuId в sites.json
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

    echo json_encode(['ok'=>true,'topMenuId'=>$menuId], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

## 2) api.php — добавить action `menu.delete`

Вставь **в секцию MENU** (лучше после `menu.update` или после `menu.create`):

```php
// menu.delete (EDITOR+): удалить меню целиком
if ($action === 'menu.delete') {
    $siteId = (int)($_POST['siteId'] ?? 0);
    $menuId = (int)($_POST['menuId'] ?? 0);

    if ($siteId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'SITE_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($menuId <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'MENU_ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    sb_require_editor($siteId);

    // удаляем меню из menus.json
    $all = sb_read_menus();
    $rec = sb_menu_get_site_record($all, $siteId);
    if (!$rec) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $menus = $rec['menus'] ?? [];
    $before = count($menus);
    $menus = array_values(array_filter($menus, fn($m) => (int)($m['id'] ?? 0) !== $menuId));
    if (count($menus) === $before) {
        http_response_code(404);
        echo json_encode(['ok'=>false,'error'=>'MENU_NOT_FOUND'], JSON_UNESCAPED_UNICODE);
        exit;
    }

    $rec['menus'] = $menus;
    sb_menu_upsert_site_record($all, $siteId, $rec);
    sb_write_menus($all);

    // если это меню было верхним — сбрасываем topMenuId
    $sites = sb_read_sites();
    foreach ($sites as &$s) {
        if ((int)($s['id'] ?? 0) === $siteId) {
            $top = (int)($s['topMenuId'] ?? 0);
            if ($top === $menuId) {
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

---

## 3) menu.php — UI: кнопка “Удалить меню” + обработчик

### 3.1. В render() добавь кнопку рядом с “Переименовать”

В твоём блоке кнопок меню (там где `data-menu-rename` и `data-item-add`) добавь:

```js
<button class="ui-btn ui-btn-danger ui-btn-xs" data-menu-delete="${m.id}">Удалить меню</button>
```

Например так:

```js
<div class="btns">
  ...кнопка "Сделать верхним"...
  <button class="ui-btn ui-btn-light ui-btn-xs" data-menu-rename="${m.id}">Переименовать</button>
  <button class="ui-btn ui-btn-primary ui-btn-xs" data-item-add="${m.id}">+ Пункт</button>
  <button class="ui-btn ui-btn-danger ui-btn-xs" data-menu-delete="${m.id}">Удалить меню</button>
</div>
```

### 3.2. В document.addEventListener('click'...) добавь обработчик

После обработчика `data-menu-rename` (или где удобно), добавь:

```js
const md = e.target.closest('[data-menu-delete]');
if (md) {
  const menuId = parseInt(md.getAttribute('data-menu-delete'), 10);
  BX.UI.Dialogs.MessageBox.show({
    title: 'Удалить меню #' + menuId + '?',
    message: 'Меню будет удалено навсегда. Продолжить?',
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      api('menu.delete', { menuId }).then(res => {
        if (!res || res.ok !== true) {
          BX.UI.Notification.Center.notify({ content: 'Не удалось удалить меню (нужен EDITOR+)' });
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

## 4) Что проверить после этих пунктов

1. Создай 2 меню → назначь одно верхним → открой `view.php` → должно показывать **именно верхнее**.
2. Удали верхнее меню → `topMenuId` должен стать 0 → `view.php` должен взять **первое меню** (fallback).
3. Назначь верхним другое меню → снова `view.php` показывает его.

---

Если всё ок — следующий “серьёзный” блок после меню:
**страницы: HomePage + роут `/{siteSlug}` на homePageId + `/{siteSlug}/{pageSlug}` на страницы**, но без rewrite можно сделать через `router.php?site=...` или просто `view.php` как сейчас.
