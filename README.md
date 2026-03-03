Ок, делаем **применение настроек из `sites.json` в `view.php`**:

* `containerWidth` → ограничение ширины контента + центрирование
* `accent` → цвет кнопок/активных элементов/ссылок
* `logoFileId` → логотип в шапке (если задан)

Ниже даю **точечные вставки** (чтобы не пересылать весь файл).

---

## 1) PHP: достаём settings (вверху, после `$topMenuId`)

Найди у себя в `view.php` строку:

```php
$topMenuId = (int)($site['topMenuId'] ?? 0);
```

СРАЗУ ПОСЛЕ неё вставь:

```php
$settings = (isset($site['settings']) && is_array($site['settings'])) ? $site['settings'] : [];

$containerWidth = (int)($settings['containerWidth'] ?? 1100);
if ($containerWidth < 900) $containerWidth = 900;
if ($containerWidth > 1600) $containerWidth = 1600;

$accent = (string)($settings['accent'] ?? '#2563eb');
if (!preg_match('~^#[0-9a-fA-F]{6}$~', $accent)) $accent = '#2563eb';

$logoFileId = (int)($settings['logoFileId'] ?? 0);
```

---

## 2) CSS: добавляем переменные и контейнер

В `<style>` в `view.php` (в начале стилей) добавь:

```css
:root{
  --sb-accent: <?= h($accent) ?>;
  --sb-container: <?= (int)$containerWidth ?>px;
}
```

### Затем замени/добавь стили под контейнер и шапку

Найди `.top` и `.content` и **обнови/дополни** так:

```css
.top {
  background:#fff;
  border-bottom:1px solid #e5e7ea;
  padding:12px 16px;
  display:flex;
  gap:10px;
  align-items:center;
  flex-wrap:wrap;
}

/* новый контейнер */
.inner {
  width: min(var(--sb-container), calc(100% - 32px));
  margin: 0 auto;
}

/* делаем top “двухслойным”: фон на всю ширину, контент по контейнеру */
.topInner{
  display:flex;
  gap:10px;
  align-items:center;
  flex-wrap:wrap;
}

/* контент тоже в контейнер */
.content { padding: 18px 0; }  /* было padding:18px; */
```

### Accent применим к ссылкам/кнопкам/активным пунктам меню

Замени (или просто добавь ниже) эти правила:

```css
a { color: var(--sb-accent); text-decoration:none; }
a:hover { text-decoration:underline; }

.menu a.active { background: color-mix(in srgb, var(--sb-accent) 12%, #fff); font-weight:bold; }
.sectionNavList a.active { background: color-mix(in srgb, var(--sb-accent) 12%, #fff); border-color: color-mix(in srgb, var(--sb-accent) 30%, #e5e7ea); font-weight:700; }

.btn-primary { background: var(--sb-accent); color:#fff; border-color: var(--sb-accent); }
```

> `color-mix` работает в современных браузерах. Если хочешь 100% совместимость — скажи, сделаю без него.

---

## 3) HTML: оборачиваем шапку и контент в `.inner`

### 3.1. Шапка

Сейчас у тебя примерно так:

```php
<div class="top">
  ...вся шапка...
</div>
```

Сделай так (важно: добавили `inner` и `topInner`):

```php
<div class="top">
  <div class="inner topInner">

    <?php if ($logoFileId > 0): ?>
      <a href="<?= h(viewUrl($siteId, (int)$pageId)) ?>" style="display:inline-flex;align-items:center;gap:10px;">
        <img src="<?= h(downloadUrl($siteId, $logoFileId)) ?>" alt="logo"
             style="height:28px;width:auto;display:block;border-radius:8px;border:1px solid #eee;background:#fff;">
      </a>
    <?php endif; ?>

    <a href="/local/sitebuilder/index.php">← Назад</a>
    <div class="muted">/</div>
    <div><b><?=h($site['name'])?></b></div>
    <div class="muted">/</div>
    <div><?=h($page['title'])?></div>

    <div style="flex:1;"></div>

    <a href="/local/sitebuilder/editor.php?siteId=<?= (int)$siteId ?>&pageId=<?= (int)$pageId ?>" target="_blank">Редактор</a>
    <a href="/local/sitebuilder/menu.php?siteId=<?= (int)$siteId ?>" target="_blank">Меню</a>

    <div style="flex-basis:100%; height:0;"></div>

    <?php if ($breadcrumbs && count($breadcrumbs) > 1): ?>
      ... breadcrumbs ...
    <?php endif; ?>

    <div style="flex-basis:100%; height:0;"></div>

    <?php if ($menuItems): ?>
      ... menu ...
    <?php else: ?>
      ... fallback menu ...
    <?php endif; ?>

  </div>
</div>
```

### 3.2. Контент

Сейчас у тебя:

```php
<div class="content">
  <div class="card"> ... </div>
</div>
```

Сделай так:

```php
<div class="content">
  <div class="inner">
    <div class="card">
      ...
    </div>
  </div>
</div>
```

---

## 4) Проверка после правок (быстро)

1. Открой `settings.php` → поменяй **Accent** на другой (например `#e11d48`) → сохрани
2. Открой `view.php` страницы → активный пункт меню и кнопки должны стать новым цветом
3. Поменяй **containerWidth** (например 1000/1400) → контент должен становиться уже/шире
4. Выбери **logoFileId** → логотип появится слева в шапке

---

Если ок — следующим шагом делаем то же самое в **`public.php`** (там самое важное: публичный доступ + применять `topMenuId`/settings без авторизации).
