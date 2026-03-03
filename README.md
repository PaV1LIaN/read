Понял. **“3 пункт” (HTML-обёртки `.inner` / `.topInner` и логотип)** нужно вставлять **внутри `<body>` там, где у тебя сейчас шапка**.

Но в твоём текущем `view.php` шапка уже переделана на:

```html
<div class="topWrap">
  <div class="container top">
    <div class="topRow"> ... </div>
    ...
  </div>
</div>
```

То есть ты взял другой вариант (с `.container`), а “3 пункт” был для варианта со `.top` + `.inner`.

### Поэтому ответ такой:

**Либо ты остаёшься на варианте `.container` (как сейчас) — тогда 3 пункт НЕ вставляешь вообще.**
**Либо хочешь именно вариант `.inner` (по `containerWidth`) — тогда нужно заменить шапку и контент.**

Сейчас у тебя проблема:
Ты уже добавил переменные `--sb-container`, но `.container` у тебя всё равно ограничен `--container: 1100px`, а не `--sb-container`. Поэтому визуально “containerWidth” может не применяться.

---

# Правильное решение для твоего текущего файла (минимально и без переписывания)

## A) Просто привяжи `.container` к настройке `--sb-container`

В CSS найди:

```css
.container{
  max-width: var(--container);
  margin: 0 auto;
  padding: 0 16px;
}
```

Замени на:

```css
.container{
  max-width: var(--sb-container);
  margin: 0 auto;
  padding: 0 16px;
}
```

И всё — **containerWidth начнёт работать** без “3 пункта”.

---

## B) Добавляем логотип (logoFileId) в твою текущую шапку

В HTML шапки найди блок:

```php
<div class="brand">
  <div class="brandMark">SB</div>
  <div class="brandTxt">
    <div class="siteName"><?=h($site['name'])?></div>
    <div class="pageName"><?=h($page['title'])?></div>
  </div>
</div>
```

Сделай так (добавил `<img>` ДО `brandMark`):

```php
<div class="brand">
  <?php if ($logoFileId > 0): ?>
    <img
      src="<?= h(downloadUrl($siteId, $logoFileId)) ?>"
      alt="logo"
      style="height:34px;width:auto;display:block;border-radius:12px;border:1px solid #eee;background:#fff;"
    >
  <?php endif; ?>

  <div class="brandMark">SB</div>
  <div class="brandTxt">
    <div class="siteName"><?=h($site['name'])?></div>
    <div class="pageName"><?=h($page['title'])?></div>
  </div>
</div>
```

---

## C) Accent: у тебя сейчас есть конфликт переменных

Ты объявил:

* `--accent:#2563eb;`
* и `--sb-accent: <?= $accent ?>;`

Но часть стилей продолжает использовать `--accent`, часть — `--sb-accent`.

Чтобы было единообразно и “accent” реально влиял на UI, сделай так:

1. В `:root` замени строку:

```css
--accent:#2563eb;
```

на:

```css
--accent: var(--sb-accent);
```

2. Тогда все места где используется `var(--accent)` тоже будут брать значение из настроек.

---

# Итог

✅ **3 пункт вставлять НЕ надо в твоём текущем файле**, потому что ты уже используешь другой макет.
Нужно только:

1. `.container { max-width: var(--sb-container); }`
2. логотип вставить в `.brand`
3. `--accent: var(--sb-accent);` чтобы цвет реально применялся

---

Если ок — после этого переходим к **settings.php + `api.site.update`** (и туда же добавим загрузку logoFileId через Disk + выбор accent и containerWidth).
