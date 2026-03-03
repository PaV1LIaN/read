Ок, тогда двигаемся по **пункту 1: “связать settings с реальным видом + улучшить UX”** — чтобы настройки **точно влияли на view.php** и были доступны из index.php.

---

## 1) index.php — добавь кнопку “Настройки” на карточке сайта

В твоём `renderSitesCards()` найди блок кнопок:

```js
<div class="siteBtns">
  <button ...>Страницы</button>
  <a ...>Меню</a>
  <a ...>Файлы</a>
  <button ...>Доступы</button>
  <button ...>Удалить</button>
</div>
```

И вставь **между “Страницы” и “Меню”**:

```html
<a class="ui-btn ui-btn-light ui-btn-xs"
   href="/local/sitebuilder/settings.php?siteId=${s.id}">
  Настройки
</a>
```

---

## 2) view.php — применяем containerWidth + показываем logoFileId в шапке

### 2.1. Контейнер ширины из settings

У тебя в `:root` уже есть:

```css
--sb-container: <?= (int)$containerWidth ?>px;
```

Но `.container` у тебя сейчас привязан к `--container`, из-за этого настройка может **не влиять**.

В CSS найди:

```css
.container{
  max-width: var(--container);
  margin: 0 auto;
  padding: 0 16px;
}
```

И замени на:

```css
.container{
  max-width: var(--sb-container);
  margin: 0 auto;
  padding: 0 16px;
}
```

*(можешь вообще убрать `--container`, чтобы не было путаницы)*

---

### 2.2. Логотип в “brandMark”

У тебя уже есть `$logoFileId`, но он не используется.
Найди HTML в шапке:

```php
<div class="brandMark">SB</div>
```

Замени на:

```php
<div class="brandMark">
  <?php if ($logoFileId > 0): ?>
    <img src="<?= h(downloadUrl($siteId, $logoFileId)) ?>" alt="logo">
  <?php else: ?>
    SB
  <?php endif; ?>
</div>
```

И добавь/замени стили для `.brandMark` (чтобы картинка красиво влезала):

```css
.brandMark{
  width: 34px;
  height: 34px;
  border-radius: 12px;
  background: var(--accentSoft);
  border:1px solid #c7d2fe;
  display:flex;
  align-items:center;
  justify-content:center;
  color: var(--accent);
  font-weight: 800;
  flex:0 0 auto;
  overflow:hidden; /* важно для img */
}
.brandMark img{
  width:100%;
  height:100%;
  object-fit:cover;
  display:block;
}
```

---

### 2.3. Ссылка на настройки прямо из view.php (удобно)

В блоке `.actions` (где “← К списку / Меню / Редактор”) добавь кнопку:

```php
<a class="btn" href="/local/sitebuilder/settings.php?siteId=<?= (int)$siteId ?>" target="_blank">Настройки</a>
```

Например так:

```php
<div class="actions">
  <a class="btn" href="/local/sitebuilder/index.php">← К списку</a>
  <a class="btn" href="/local/sitebuilder/settings.php?siteId=<?= (int)$siteId ?>" target="_blank">Настройки</a>
  <a class="btn" href="/local/sitebuilder/menu.php?siteId=<?= (int)$siteId ?>" target="_blank">Меню</a>
  <a class="btn btnPrimary" href="/local/sitebuilder/editor.php?siteId=<?= (int)$siteId ?>&pageId=<?= (int)$pageId ?>" target="_blank">Редактор</a>
</div>
```

---

## Как проверить, что всё реально работает

1. Открой `settings.php` → поменяй **containerWidth** (например 900 или 1600) → **Сохранить**
2. Открой `view.php` — ширина должна заметно измениться
3. Выбери логотип (logoFileId) → Сохранить → в `view.php` в левом квадратике должен появиться логотип

---

Если сделал — дальше логичный следующий шаг: **добавить “быстрый предпросмотр” настроек в index.php (цвет/логотип) + мелкую валидацию на UI**, чтобы на серьёзном проекте меньше ловить ошибки на API.
