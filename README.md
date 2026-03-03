Ок, для серьёзного проекта делаем так, как обычно делают нормальные CMS:

1. **Вложенность = структура сайта** (контентная и навигационная модель).
2. **Меню = отдельная сущность** (может совпадать со структурой, а может нет).
3. В публичной части вложенность должна давать пользователю:

   * **хлебные крошки** (где я нахожусь),
   * **навигацию по разделу** (дети текущей страницы, и/или “соседи”).

Без rewrite/роутинга мы всё равно можем дать “ощущение сайта”.

Ниже — готовые изменения для `view.php` (вставляй как есть). Я делаю максимально аккуратно: без ломания текущего меню, без новых API, чисто по `pages.json`.

---

## 1) Вставь функции после `viewUrl()` (вверху view.php)

Найди в `view.php` блок где у тебя:

```php
function viewUrl(int $siteId, int $pageId): string {
    return '/local/sitebuilder/view.php?siteId=' . $siteId . '&pageId=' . $pageId;
}
```

Сразу **после него** вставь:

```php
function sb_index_pages_by_id(array $pages, int $siteId): array {
    $byId = [];
    foreach ($pages as $p) {
        if ((int)($p['siteId'] ?? 0) !== $siteId) continue;
        $id = (int)($p['id'] ?? 0);
        if ($id > 0) $byId[$id] = $p;
    }
    return $byId;
}

function sb_build_breadcrumbs(array $pagesById, int $currentPageId): array {
    // возвращает массив страниц от корня к текущей (включая текущую)
    $chain = [];
    $guard = 0;
    $curId = $currentPageId;

    while ($curId > 0 && isset($pagesById[$curId])) {
        $chain[] = $pagesById[$curId];
        $parentId = (int)($pagesById[$curId]['parentId'] ?? 0);
        if ($parentId <= 0) break;
        $curId = $parentId;

        $guard++;
        if ($guard > 50) break; // защита от циклов
    }

    return array_reverse($chain);
}

function sb_get_children_pages(array $pages, int $siteId, int $parentId): array {
    $kids = array_values(array_filter($pages, function($p) use ($siteId, $parentId){
        return (int)($p['siteId'] ?? 0) === $siteId && (int)($p['parentId'] ?? 0) === $parentId;
    }));
    usort($kids, function($a, $b){
        $sa = (int)($a['sort'] ?? 500);
        $sb = (int)($b['sort'] ?? 500);
        if ($sa === $sb) return ((int)($a['id'] ?? 0) <=> (int)($b['id'] ?? 0));
        return $sa <=> $sb;
    });
    return $kids;
}
```

---

## 2) Подготовь данные (после `$sitePages`)

У тебя уже есть:

```php
$sitePages = array_values(array_filter($pages, fn($p) => (int)($p['siteId'] ?? 0) === $siteId));
usort($sitePages, fn($a, $b) => (int)($a['sort'] ?? 500) <=> (int)($b['sort'] ?? 500));
```

Сразу **после этого** вставь:

```php
$pagesById = sb_index_pages_by_id($pages, $siteId);
$breadcrumbs = sb_build_breadcrumbs($pagesById, $pageId);
$children = sb_get_children_pages($pages, $siteId, $pageId);
$parentId = (int)($page['parentId'] ?? 0);
$parentChildren = $parentId > 0 ? sb_get_children_pages($pages, $siteId, $parentId) : [];
```

---

## 3) Добавь стили (в `<style>` view.php)

В твой `<style>` добавь в конец:

```css
.breadcrumbs { display:flex; gap:6px; flex-wrap:wrap; align-items:center; }
.breadcrumbs a, .breadcrumbs span { font-size:12px; color:#6a737f; }
.breadcrumbs a { color:#0b57d0; }
.breadcrumbs .sep { color:#9aa3af; }

.sectionNav {
  margin-top:14px;
  border:1px solid #e5e7ea;
  border-radius:14px;
  padding:12px;
  background:#fff;
}
.sectionNavTitle { font-weight:700; font-size:13px; }
.sectionNavList { margin-top:10px; display:flex; gap:8px; flex-wrap:wrap; }
.sectionNavList a {
  display:inline-block;
  padding:6px 10px;
  border-radius:999px;
  border:1px solid #e5e7ea;
  background:#fff;
  text-decoration:none;
}
.sectionNavList a.active { background:#eef2ff; border-color:#c7d2fe; font-weight:700; }
.sectionNavHint { margin-top:8px; font-size:12px; color:#6a737f; }
```

---

## 4) Выведи хлебные крошки в шапке (в `.top`)

Найди в `view.php` блок `.top`, где ты выводишь название сайта/страницы и ниже меню.

Перед меню (или сразу после ссылки “Меню”) вставь:

```php
<div style="flex-basis:100%; height:0;"></div>

<?php if ($breadcrumbs && count($breadcrumbs) > 1): ?>
  <div class="breadcrumbs">
    <?php foreach ($breadcrumbs as $i => $bc): ?>
      <?php
        $isLast = ($i === count($breadcrumbs)-1);
        $bcId = (int)($bc['id'] ?? 0);
        $bcTitle = (string)($bc['title'] ?? ('page#'.$bcId));
      ?>
      <?php if ($i > 0): ?><span class="sep">/</span><?php endif; ?>
      <?php if ($isLast): ?>
        <span><?= h($bcTitle) ?></span>
      <?php else: ?>
        <a href="<?= h(viewUrl($siteId, $bcId)) ?>"><?= h($bcTitle) ?></a>
      <?php endif; ?>
    <?php endforeach; ?>
  </div>
<?php endif; ?>
```

⚠️ Если у тебя уже есть `flex-basis:100%` до меню — не дублируй. Оставь один.

---

## 5) Добавь “навигацию по разделу” под заголовком страницы

Внутри `.card`, сразу после:

```php
<h1 style="margin-top:0;"><?=h($page['title'])?></h1>
```

вставь:

```php
<?php
  // что показываем:
  // 1) если у текущей страницы есть дети -> показываем “Страницы раздела”
  // 2) иначе, если у страницы есть родитель -> показываем “В этом разделе” (соседи)
?>
<?php if ($children && count($children) > 0): ?>
  <div class="sectionNav">
    <div class="sectionNavTitle">Страницы раздела</div>
    <div class="sectionNavList">
      <?php foreach ($children as $ch): ?>
        <?php $chId = (int)($ch['id'] ?? 0); ?>
        <a href="<?= h(viewUrl($siteId, $chId)) ?>">
          <?= h((string)($ch['title'] ?? ('page#'.$chId))) ?>
        </a>
      <?php endforeach; ?>
    </div>
    <div class="sectionNavHint">Раздел — это страница, у которой есть дочерние страницы (parentId = <?= (int)$pageId ?>).</div>
  </div>
<?php elseif ($parentId > 0 && $parentChildren && count($parentChildren) > 0): ?>
  <div class="sectionNav">
    <div class="sectionNavTitle">В этом разделе</div>
    <div class="sectionNavList">
      <?php foreach ($parentChildren as $ch): ?>
        <?php $chId = (int)($ch['id'] ?? 0); $active = ($chId === $pageId); ?>
        <a class="<?= $active ? 'active' : '' ?>" href="<?= h(viewUrl($siteId, $chId)) ?>">
          <?= h((string)($ch['title'] ?? ('page#'.$chId))) ?>
        </a>
      <?php endforeach; ?>
    </div>
    <div class="sectionNavHint">Ты внутри раздела (parentId = <?= (int)$parentId ?>). Тут показаны “соседи”.</div>
  </div>
<?php endif; ?>
```

---

# Что получится по UX

* В админке — дерево и “вложить”.
* В публичной части:

  * видны **крошки** (цепочка родителей),
  * если это раздел — показаны **дети**,
  * если это дочерняя — показаны **соседи** в разделе.

Это и есть “правильно” для серьёзного проекта на текущем этапе: **структура работает**, но меню остаётся управляемым вручную.

---

## Дальше (следующий шаг проекта)

После этого логичный “профессиональный” шаг — сделать **выбор topMenuId у сайта** (как ты и предложил) и переключить view.php, чтобы он рендерил **не “первое меню”, а выбранное**.

Готов двигаться: скажи “делаем topMenuId”, и я дам точные правки:

* `sites.json` структура
* `api.php`: `site.update` или отдельное `site.setTopMenu`
* `menu.php`: кнопка “Сделать верхним”
* `view.php`: брать именно topMenuId
