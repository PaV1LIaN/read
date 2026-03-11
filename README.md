Начинаем с самого важного для публичной части:

## Что сейчас делаем

Доводим `public.php` до правильной логики публикации:

* `draft` страницы не открываются снаружи,
* если запрошен `slug` черновика — отдаём 404,
* home page берётся только если она `published`,
* fallback на первую страницу — тоже только среди `published`,
* меню не должно вести на `draft` страницы.

---

# Что нужно изменить в `public.php`

## 1. Добавить функцию проверки опубликованности страницы

Рядом с `find_page()` и `find_page_by_slug()` добавь:

```php
function is_page_published(array $page): bool {
    $status = strtolower((string)($page['status'] ?? 'published'));
    return $status === 'published';
}
```

---

## 2. Исправить `find_page_by_slug()`

Сейчас публичка может находить любую страницу по slug.
Нужно, чтобы находила только опубликованную.

### Было:

```php
function find_page_by_slug(int $siteId, string $slug): ?array {
    foreach (sb_read_pages() as $p) {
        if ((int)($p['siteId'] ?? 0) === $siteId && (string)($p['slug'] ?? '') === $slug) return $p;
    }
    return null;
}
```

### Должно стать:

```php
function find_page_by_slug(int $siteId, string $slug): ?array {
    foreach (sb_read_pages() as $p) {
        if ((int)($p['siteId'] ?? 0) !== $siteId) continue;
        if ((string)($p['slug'] ?? '') !== $slug) continue;
        if (!is_page_published($p)) continue;
        return $p;
    }
    return null;
}
```

---

## 3. Исправить `find_page()`

Чтобы через `homePageId` случайно не открывался draft.

### Было:

```php
function find_page(int $pageId): ?array {
    foreach (sb_read_pages() as $p) if ((int)($p['id'] ?? 0) === $pageId) return $p;
    return null;
}
```

### Должно стать:

```php
function find_page(int $pageId): ?array {
    foreach (sb_read_pages() as $p) {
        if ((int)($p['id'] ?? 0) === $pageId) return $p;
    }
    return null;
}
```

Оставляем так, но ниже в логике выбора страницы будем отдельно проверять `is_page_published()`.

---

## 4. Исправить выбор текущей страницы

Вот этот блок:

```php
$page = null;
if ($slug !== '') $page = find_page_by_slug($siteId, $slug);

if (!$page) {
    $homeId = (int)($site['homePageId'] ?? 0);
    if ($homeId > 0) $page = find_page($homeId);
}
if (!$page) {
    // если home не задан — берём первую страницу
    foreach (sb_read_pages() as $p) {
        if ((int)($p['siteId'] ?? 0) === $siteId) { $page = $p; break; }
    }
}
if (!$page) { http_response_code(404); echo 'Page not found'; exit; }
```

### Замени на:

```php
$page = null;

// 1. Явный slug — только published
if ($slug !== '') {
    $page = find_page_by_slug($siteId, $slug);
    if (!$page) {
        http_response_code(404);
        echo 'Page not found';
        exit;
    }
}

// 2. Home page — только published
if (!$page) {
    $homeId = (int)($site['homePageId'] ?? 0);
    if ($homeId > 0) {
        $candidate = find_page($homeId);
        if ($candidate && (int)($candidate['siteId'] ?? 0) === $siteId && is_page_published($candidate)) {
            $page = $candidate;
        }
    }
}

// 3. Fallback — первая опубликованная страница сайта
if (!$page) {
    $publishedPages = array_values(array_filter(sb_read_pages(), function($p) use ($siteId) {
        return (int)($p['siteId'] ?? 0) === $siteId && is_page_published($p);
    }));

    usort($publishedPages, fn($a, $b) =>
        ((int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500))
        ?: ((int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0))
    );

    if ($publishedPages) {
        $page = $publishedPages[0];
    }
}

if (!$page) {
    http_response_code(404);
    echo 'Page not found';
    exit;
}
```

---

## 5. Исправить формирование меню

Сейчас в меню могут попадать ссылки на draft-страницы.
Нужно фильтровать их.

Найди блок внутри:

```php
if ($type === 'page') {
    $pid = (int)($it['pageId'] ?? 0);
    $p = $pid > 0 ? find_page($pid) : null;
    ...
}
```

### Замени на:

```php
if ($type === 'page') {
    $pid = (int)($it['pageId'] ?? 0);
    $p = $pid > 0 ? find_page($pid) : null;

    if ($p && (int)($p['siteId'] ?? 0) === $siteId && is_page_published($p)) {
        $href = public_page_url($siteId, $p);
        if ($title === 'Link') $title = (string)($p['title'] ?? 'Page');
    } else {
        continue;
    }
}
```

---

# Что получится после этого

После правки:

* если страница `draft`, она не откроется по `public.php?...&p=slug`
* если `homePageId` указывает на черновик, он будет проигнорирован
* если опубликованных страниц нет вообще — будет 404
* меню не покажет ссылки на черновики

---

# Как проверить

1. Сделай одну страницу `published`, вторую `draft`
2. Открой:

   * опубликованную через `public.php?siteId=...&p=...`
   * черновик через `public.php?siteId=...&p=...`
3. У опубликованной страница должна открыться
4. У черновика должен быть `404 Page not found`
5. Если `homePageId` указывает на draft, публичка должна открыть первую опубликованную страницу
6. В меню draft-страница не должна отображаться

Следующим шагом после этого я бы сделал, чтобы `public.php` использовал не “первое меню”, а именно `topMenuId`, как уже сделано в `view.php`.
