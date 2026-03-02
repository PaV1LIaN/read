Ок, публикацию откладываем. Тогда самый полезный следующий шаг “как у тильды”, но без роутинга и без билдов — **структура страниц (дерево) + drag&drop сортировка + вложенность**. Это сразу делает проект “сайтостроителем”, а не просто списком страниц.

Сейчас у тебя у страниц уже есть поля `parentId` и `sort`, но мы ими почти не пользуемся. Давай их оживим:

## Что делаем следующим шагом (MVP)

1. В `index.php` (где список страниц) показываем **дерево страниц**:

   * корневые страницы
   * вложенные (parentId)
2. Добавляем действия:

   * **переместить страницу вверх/вниз** среди соседей
   * **сделать дочерней / сделать корневой**
   * **переименовать**
3. В `api.php` добавляем 3 actions:

   * `page.updateMeta` (title/slug/sort/parentId)
   * `page.move` (up/down)
   * `page.setParent` (parentId)

Это даст тебе:

* меню сайта можно будет собирать по дереву (очень скоро)
* навигация по сайту будет логичной
* появляется “структура сайта”, как в тильде

---

# 1) api.php — добавляем actions (готовый код)

Вставь **перед UNKNOWN_ACTION**.

### A) `page.updateMeta` (переименование/slug)

```php
if ($action === 'page.updateMeta') {
    $id = (int)($_POST['id'] ?? 0);
    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($id);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $title = trim((string)($_POST['title'] ?? ''));
    $slugIn = trim((string)($_POST['slug'] ?? ''));

    $pages = sb_read_pages();
    $found = false;

    // slug (если дали)
    $newSlug = $slugIn !== '' ? sb_slugify($slugIn) : (string)($page['slug'] ?? 'page-'.$id);
    if ($title !== '' && $slugIn === '') {
        // если slug не задан, но title изменили — можем пересчитать slug от title (аккуратно)
        $newSlug = sb_slugify($title);
    }

    // уникальность slug в рамках siteId
    $existing = array_map(fn($x) => (string)($x['slug'] ?? ''),
        array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId && (int)($p['id'] ?? 0) !== $id)
    );
    $base = $newSlug; $i = 2;
    while (in_array($newSlug, $existing, true)) { $newSlug = $base.'-'.$i; $i++; }

    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            if ($title !== '') $p['title'] = $title;
            $p['slug'] = $newSlug;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            $found = true;
            break;
        }
    }
    unset($p);

    if (!$found) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    sb_write_pages($pages);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

### B) `page.setParent` (вложенность)

```php
if ($action === 'page.setParent') {
    $id = (int)($_POST['id'] ?? 0);
    $parentId = (int)($_POST['parentId'] ?? 0); // 0 = корневая

    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($id);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    if ($parentId > 0) {
        $parent = sb_find_page($parentId);
        if (!$parent || (int)($parent['siteId'] ?? 0) !== $siteId) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'PARENT_NOT_IN_SITE'], JSON_UNESCAPED_UNICODE);
            exit;
        }
        if ($parentId === $id) {
            http_response_code(422);
            echo json_encode(['ok'=>false,'error'=>'PARENT_SELF'], JSON_UNESCAPED_UNICODE);
            exit;
        }
    }

    $pages = sb_read_pages();
    foreach ($pages as &$p) {
        if ((int)($p['id'] ?? 0) === $id) {
            $p['parentId'] = $parentId;
            $p['updatedAt'] = date('c');
            $p['updatedBy'] = (int)$USER->GetID();
            break;
        }
    }
    unset($p);

    sb_write_pages($pages);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

### C) `page.move` (up/down среди соседей)

```php
if ($action === 'page.move') {
    $id = (int)($_POST['id'] ?? 0);
    $dir = (string)($_POST['dir'] ?? '');

    if ($id <= 0) { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'ID_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir !== 'up' && $dir !== 'down') { http_response_code(422); echo json_encode(['ok'=>false,'error'=>'DIR_REQUIRED'], JSON_UNESCAPED_UNICODE); exit; }

    $page = sb_find_page($id);
    if (!$page) { http_response_code(404); echo json_encode(['ok'=>false,'error'=>'PAGE_NOT_FOUND'], JSON_UNESCAPED_UNICODE); exit; }

    $siteId = (int)($page['siteId'] ?? 0);
    sb_require_editor($siteId);

    $parentId = (int)($page['parentId'] ?? 0);

    $pages = sb_read_pages();
    $siblings = array_values(array_filter($pages, fn($p) =>
        (int)($p['siteId'] ?? 0) === $siteId && (int)($p['parentId'] ?? 0) === $parentId
    ));
    usort($siblings, fn($a,$b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));

    $pos = null;
    for ($i=0; $i<count($siblings); $i++) {
        if ((int)($siblings[$i]['id'] ?? 0) === $id) { $pos = $i; break; }
    }
    if ($pos === null) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }

    if ($dir === 'up' && $pos === 0) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }
    if ($dir === 'down' && $pos === count($siblings)-1) { echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE); exit; }

    $swap = ($dir === 'up') ? $pos-1 : $pos+1;

    $idA = (int)$siblings[$pos]['id'];
    $idB = (int)$siblings[$swap]['id'];
    $sortA = (int)($siblings[$pos]['sort'] ?? 500);
    $sortB = (int)($siblings[$swap]['sort'] ?? 500);

    foreach ($pages as &$p) {
        $pid = (int)($p['id'] ?? 0);
        if ($pid === $idA) { $p['sort'] = $sortB; $p['updatedAt']=date('c'); $p['updatedBy']=(int)$USER->GetID(); }
        if ($pid === $idB) { $p['sort'] = $sortA; $p['updatedAt']=date('c'); $p['updatedBy']=(int)$USER->GetID(); }
    }
    unset($p);

    sb_write_pages($pages);
    echo json_encode(['ok'=>true], JSON_UNESCAPED_UNICODE);
    exit;
}
```

---

# 2) Дальше: правим index.php (страницы как дерево)

Чтобы я прислал **точно подходящий** готовый `index.php`, мне нужно увидеть твой текущий `index.php` (где список сайтов/страниц сейчас).

✅ Пришли содержимое `/local/sitebuilder/index.php`, и я:

* добавлю дерево страниц,
* добавлю кнопки ↑ ↓, “вложить”, “в корень”, “переименовать”,
* и всё будет работать с новыми actions без лишней логики.
