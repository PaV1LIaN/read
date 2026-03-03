Понял. Твой текущий `public.php` — это **полноценный рендерер** (он сам читает JSON, сам строит меню, сам рисует блоки). Для «серьёзного проекта» у него две проблемы:

1. **Дублирование логики** `view.php` → через месяц они разъедутся (стили, настройки, меню `topMenuId`, breadcrumbs, новые блоки и т.д.).
2. **Самое важное:** он сейчас **без авторизации/ACL**. Это значит, что любой, кто знает `siteId`, может открыть сайт (и часто ещё и картинки через `download.php`, если там нет жёстких проверок). Это потенциальная дырка.

Поэтому правильнее сделать так:

* `public.php` = **входная точка / роутер**: выбирает страницу (по `p`/`pageId`/`homePageId`/первой корневой) и **редиректит** на рендерер.
* Рендерер один:

  * сейчас можно редиректить на `view.php` (пока public не настоящий),
  * позже сделаем отдельный `public_view.php` (без авторизации, но с токеном/флагом).

Ниже — **вариант “как надо сейчас”**, и он **сохраняет совместимость** с твоими ссылками `public.php?siteId=...&p=slug`.

---

## ✅ Что делать с твоим старым public.php

1. **Переименуй его**, например в:
   `/local/sitebuilder/public_legacy_render.php`
   (чтобы не потерять код)

2. **Создай новый** `/local/sitebuilder/public.php` (роутер) вот такой:

```php
<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

global $USER;

// Пока public не “по-настоящему публичный” — держим под авторизацией
if (!$USER->IsAuthorized()) {
    LocalRedirect('/auth/');
}

$siteId = (int)($_GET['siteId'] ?? 0);

// совместимость со старым вариантом:
// public.php?siteId=1&p=about
$slug = trim((string)($_GET['p'] ?? ''));

// совместимость с “прямым” открытием:
// public.php?siteId=1&pageId=10
$pageId = (int)($_GET['pageId'] ?? 0);

function sb_data_path(string $file): string {
    return $_SERVER['DOCUMENT_ROOT'] . '/upload/sitebuilder/' . $file;
}
function sb_read_json(string $file): array {
    $path = sb_data_path($file);
    if (!file_exists($path)) return [];
    $raw = file_get_contents($path);
    $data = json_decode((string)$raw, true);
    return is_array($data) ? $data : [];
}

if ($siteId <= 0) { http_response_code(404); echo 'SITE_ID_REQUIRED'; exit; }

$sites = sb_read_json('sites.json');
$pages = sb_read_json('pages.json');

$site = null;
foreach ($sites as $s) {
    if ((int)($s['id'] ?? 0) === $siteId) { $site = $s; break; }
}
if (!$site) { http_response_code(404); echo 'SITE_NOT_FOUND'; exit; }

// --- 1) Явный pageId ---
$targetPageId = 0;
if ($pageId > 0) {
    foreach ($pages as $p) {
        if ((int)($p['id'] ?? 0) === $pageId && (int)($p['siteId'] ?? 0) === $siteId) {
            $targetPageId = $pageId;
            break;
        }
    }
}

// --- 2) slug p=... ---
if ($targetPageId <= 0 && $slug !== '') {
    foreach ($pages as $p) {
        if ((int)($p['siteId'] ?? 0) === $siteId && (string)($p['slug'] ?? '') === $slug) {
            $targetPageId = (int)($p['id'] ?? 0);
            break;
        }
    }
}

// --- 3) homePageId ---
if ($targetPageId <= 0) {
    $home = (int)($site['homePageId'] ?? 0);
    if ($home > 0) {
        foreach ($pages as $p) {
            if ((int)($p['id'] ?? 0) === $home && (int)($p['siteId'] ?? 0) === $siteId) {
                $targetPageId = $home;
                break;
            }
        }
    }
}

// --- 4) первая корневая (parentId=0) по sort/id ---
if ($targetPageId <= 0) {
    $rootPages = array_values(array_filter($pages, function($p) use ($siteId){
        return (int)($p['siteId'] ?? 0) === $siteId && (int)($p['parentId'] ?? 0) === 0;
    }));

    usort($rootPages, function($a, $b){
        $sa = (int)($a['sort'] ?? 500);
        $sb = (int)($b['sort'] ?? 500);
        if ($sa === $sb) return ((int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0));
        return $sa <=> $sb;
    });

    if ($rootPages) $targetPageId = (int)($rootPages[0]['id'] ?? 0);
}

// --- 5) fallback: вообще первая страница сайта ---
if ($targetPageId <= 0) {
    $sitePages = array_values(array_filter($pages, fn($p)=> (int)($p['siteId'] ?? 0) === $siteId));
    usort($sitePages, function($a, $b){
        $sa = (int)($a['sort'] ?? 500);
        $sb = (int)($b['sort'] ?? 500);
        if ($sa === $sb) return ((int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0));
        return $sa <=> $sb;
    });
    if ($sitePages) $targetPageId = (int)($sitePages[0]['id'] ?? 0);
}

if ($targetPageId <= 0) { http_response_code(404); echo 'NO_PAGES'; exit; }

// Сейчас используем единый рендерер:
// позже поменяем на public_view.php (для настоящего публичного режима)
LocalRedirect('/local/sitebuilder/view.php?siteId='.$siteId.'&pageId='.$targetPageId);
```

---

## Что проверить

* `public.php?siteId=1` → открывает home/первую корневую
* `public.php?siteId=1&p=about` → открывает страницу по slug
* `public.php?siteId=1&pageId=10` → открывает страницу по id

---

## Следующий шаг (то, что я “хотел сделать с public”)

Чтобы это стало **реально публичным**, нам нужно:

1. `settings.php`: `publicEnabled` + `publicToken` (генерируем), кнопка “скопировать ссылку”
2. `public.php`: если `publicEnabled=1` и `token` верный → пускать без авторизации
3. `view.php` оставить внутренним (как сейчас)
4. сделать `public_view.php` (рендер без кнопок «Редактор/Меню», без авторизации, но с токеном)

Скажи: **делаем сразу “public по токену”** (пункты 1–2), или сначала просто добавим в settings “Публичная ссылка” (без отключения авторизации)?
