Ок — идём дальше без вопросов. Следующий “тильда-шаг”, который резко повышает ценность: **меню на публичной странице** (чтобы навигация работала на опубликованном сайте).

Сделаем так:

* в редакторе меню ты уже умеешь `menu.*` (у тебя в `api.php` есть menus.json)
* в **public.php** (и при желании в view.php) добавим вывод **главного меню** + ссылки на страницы по slug.

---

# 1) public.php — добавить вывод меню

## 1.1 Добавь чтение menus.json

В public.php рядом с другими `sb_read_*` добавь:

```php
function sb_read_menus(): array { return sb_read_json_file('menus.json'); }
```

## 1.2 Хелпер: получить меню сайта

Добавь функции:

```php
function sb_menu_get_site_record(array $all, int $siteId): ?array {
    foreach ($all as $r) if ((int)($r['siteId'] ?? 0) === $siteId) return $r;
    return null;
}
function sb_menu_pick_main(array $siteRecord): ?array {
    $menus = $siteRecord['menus'] ?? [];
    if (!is_array($menus) || !count($menus)) return null;
    // пока берём первое меню как "главное"
    return $menus[0];
}
```

## 1.3 Хелпер: url для пункта меню

Добавь:

```php
function public_page_url(int $siteId, ?array $page): string {
    $slug = $page ? (string)($page['slug'] ?? '') : '';
    if ($slug !== '') return '/local/sitebuilder/public.php?siteId=' . $siteId . '&p=' . urlencode($slug);
    return '/local/sitebuilder/public.php?siteId=' . $siteId;
}
```

## 1.4 Перед выводом HTML (после выбора `$page`)

Собери меню:

```php
$menuItems = [];
$allMenus = sb_read_menus();
$siteRec = sb_menu_get_site_record($allMenus, $siteId);
$mainMenu = $siteRec ? sb_menu_pick_main($siteRec) : null;

if ($mainMenu && is_array($mainMenu['items'] ?? null)) {
    $items = $mainMenu['items'];
    usort($items, fn($a,$b) => (int)($a['sort'] ?? 0) <=> (int)($b['sort'] ?? 0));

    foreach ($items as $it) {
        if (!is_array($it)) continue;
        $type = (string)($it['type'] ?? '');
        $title = trim((string)($it['title'] ?? ''));
        if ($title === '') $title = 'Link';

        $href = '#';
        if ($type === 'page') {
            $pid = (int)($it['pageId'] ?? 0);
            $p = $pid > 0 ? find_page($pid) : null;
            // безопасность: только страницы этого сайта
            if ($p && (int)($p['siteId'] ?? 0) === $siteId) {
                $href = public_page_url($siteId, $p);
                if ($title === 'Link') $title = (string)($p['title'] ?? 'Page');
            } else {
                continue;
            }
        } elseif ($type === 'url') {
            $url = trim((string)($it['url'] ?? ''));
            if ($url === '') continue;
            $href = $url;
        } else {
            continue;
        }

        $menuItems[] = ['title' => $title, 'href' => $href];
    }
}
```

## 1.5 В HTML добавь шапку меню

В `<style>` добавь:

```css
.header{position:sticky;top:0;background:#fff;border-bottom:1px solid #eee;z-index:10;}
.header .in{max-width:980px;margin:0 auto;padding:12px 24px;display:flex;gap:14px;align-items:center;flex-wrap:wrap;}
.brand{font-weight:800;}
.nav{display:flex;gap:12px;flex-wrap:wrap;}
.nav a{color:#111;text-decoration:none;padding:6px 10px;border-radius:10px;}
.nav a:hover{background:#f3f4f6;}
```

И сразу после `<body>` вставь:

```php
<div class="header">
  <div class="in">
    <div class="brand"><?=h($site['name'] ?? 'Site')?></div>
    <?php if (count($menuItems)): ?>
      <nav class="nav">
        <?php foreach ($menuItems as $mi): ?>
          <a href="<?=h($mi['href'])?>" <?=preg_match('~^https?://~i',$mi['href'])?'target="_blank" rel="noopener noreferrer"':''?>>
            <?=h($mi['title'])?>
          </a>
        <?php endforeach; ?>
      </nav>
    <?php endif; ?>
  </div>
</div>
```

И чуть ниже у тебя уже идёт `<div class="wrap">...`

---

# 2) Что проверить

1. Создай меню в редакторе (хотя бы 1 пункт на страницу)
2. Открой:
   `/local/sitebuilder/public.php?siteId=1`
3. Вверху должна появиться шапка с названием сайта и пунктами меню
4. Клик по пунктам “page” ведёт на `?p=slug`

---

# 3) Следующий шаг (после меню)

**Публикация “в один адрес”**: `/s/{siteSlug}/{pageSlug}` через rewrite + автоматический `public.php` роутинг по slug’ам (без `siteId=...`). Это уже будет максимально похоже на Tilda.

Сейчас просто сделай изменения в public.php и скажи: меню вверху появилось?
