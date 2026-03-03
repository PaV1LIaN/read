Ок, давай делаем **`public.php` как “стабильную ссылку на сайт”** (в серьёзном проекте это must-have): ты даёшь людям **один URL**, а он сам открывает **домашнюю страницу сайта** (или первую корневую, если home не задан). Никаких `pageId` руками.

## Что делает `public.php`

* Принимает `siteId`
* Определяет, какую страницу открыть:

  1. если передан `pageId` — откроет её (если она в этом site)
  2. иначе если у сайта задан `homePageId` — откроет её
  3. иначе откроет **первую корневую страницу** (`parentId=0`, по sort/id)
* Делает редирект на `view.php?siteId=...&pageId=...`
* (Пока) требует авторизацию — так безопаснее. Позже легко расширим до публичного доступа по токену/флагу.

---

## Файл `/local/sitebuilder/public.php` (создай новый)

```php
<?php
define('NO_KEEP_STATISTIC', true);
define('NO_AGENT_STATISTIC', true);
define('DisableEventsCheck', true);

require $_SERVER['DOCUMENT_ROOT'].'/bitrix/modules/main/include/prolog_before.php';

global $USER;

if (!$USER->IsAuthorized()) {
    // если у тебя другой путь логина — замени
    LocalRedirect('/auth/');
}

$siteId = (int)($_GET['siteId'] ?? 0);
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

if ($siteId <= 0) {
    http_response_code(404);
    echo 'SITE_NOT_FOUND';
    exit;
}

$sites = sb_read_json('sites.json');
$pages = sb_read_json('pages.json');

$site = null;
foreach ($sites as $s) {
    if ((int)($s['id'] ?? 0) === $siteId) { $site = $s; break; }
}
if (!$site) {
    http_response_code(404);
    echo 'SITE_NOT_FOUND';
    exit;
}

// 1) если pageId передали — проверим принадлежность сайту
$targetPageId = 0;
if ($pageId > 0) {
    foreach ($pages as $p) {
        if ((int)($p['id'] ?? 0) === $pageId && (int)($p['siteId'] ?? 0) === $siteId) {
            $targetPageId = $pageId;
            break;
        }
    }
}

// 2) иначе homePageId
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

// 3) иначе первая корневая страница
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

if ($targetPageId <= 0) {
    http_response_code(404);
    echo 'NO_PAGES';
    exit;
}

LocalRedirect('/local/sitebuilder/view.php?siteId='.$siteId.'&pageId='.$targetPageId);
```

---

## Что проверить после создания

1. В настройках сайта выстави “Домашняя страница” → сохрани
2. Открой в браузере:
   `/local/sitebuilder/public.php?siteId=1`
   Должно перекинуть на `view.php` с нужным `pageId`.

---

Если хочешь — следующим шагом сделаем **“Публичная ссылка”** в `settings.php` (просто показывать/копировать URL `public.php?siteId=...`) и кнопку “Открыть публично”. Потом уже добавим **режим без авторизации** (через `publicEnabled` + `token`).
