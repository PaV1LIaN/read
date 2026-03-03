Отличный `index.php`. Чтобы добавить **кнопку “Настройки”** правильно и аккуратно, тебе нужно сделать **2 правки**:

---

## 1) Добавь кнопку в карточку сайта (в `renderSitesCards`)

Найди блок кнопок внутри шаблона карточки:

```js
<div class="siteBtns">
  ...
</div>
```

И **вставь кнопку “Настройки”** (я вставляю её логично рядом с “Файлы/Меню”):

✅ Вставь **после “Файлы”** и **до “Доступы”** вот это:

```html
<a class="ui-btn ui-btn-light ui-btn-xs"
   href="/local/sitebuilder/settings.php?siteId=${s.id}"
   target="_blank">
  Настройки
</a>
```

То есть получится так (показываю кусок целиком, чтобы не ошибиться):

```js
<div class="siteBtns">
  <button class="ui-btn ui-btn-light ui-btn-xs"
          data-open-pages-site-id="${s.id}"
          data-open-pages-site-name="${BX.util.htmlspecialchars(safeStr(s.name))}">
    Страницы
  </button>

  <a class="ui-btn ui-btn-light ui-btn-xs"
     href="/local/sitebuilder/menu.php?siteId=${s.id}">
    Меню
  </a>

  <a class="ui-btn ui-btn-light ui-btn-xs"
     href="/local/sitebuilder/files.php?siteId=${s.id}"
     target="_blank">
    Файлы
  </a>

  <!-- ✅ ДОБАВИТЬ ВОТ ЭТО -->
  <a class="ui-btn ui-btn-light ui-btn-xs"
     href="/local/sitebuilder/settings.php?siteId=${s.id}"
     target="_blank">
    Настройки
  </a>

  <button class="ui-btn ui-btn-light ui-btn-xs"
          data-open-access-site-id="${s.id}"
          data-open-access-site-name="${BX.util.htmlspecialchars(safeStr(s.name))}">
    Доступы
  </button>

  <button class="ui-btn ui-btn-danger ui-btn-xs"
          data-delete-site-id="${s.id}">
    Удалить
  </button>
</div>
```

---

## 2) (Опционально, но правильно) Добавь “пилл” SETTINGS если есть настройки

У тебя уже есть `sitePills(s)` — можно красиво подсветить факт настроек.

Внутри `sitePills(s)` добавь после HOME:

```js
const st = (s.settings && typeof s.settings === 'object') ? s.settings : null;
if (st && (st.containerWidth || st.accent || st.logoFileId)) pills.push('<span class="pill pillStrong">SETTINGS</span>');
```

Будет так:

```js
function sitePills(s){
  const pills = [];
  if (parseInt(s.topMenuId || 0, 10) > 0) pills.push('<span class="pill pillStrong">TOP MENU</span>');
  if (parseInt(s.homePageId || 0, 10) > 0) pills.push('<span class="pill pillStrong">HOME</span>');

  const st = (s.settings && typeof s.settings === 'object') ? s.settings : null;
  if (st && (st.containerWidth || st.accent || st.logoFileId)) pills.push('<span class="pill pillStrong">SETTINGS</span>');

  pills.push('<span class="pill">ID ' + parseInt(s.id||0,10) + '</span>');
  pills.push('<span class="pill"><code>' + BX.util.htmlspecialchars(safeStr(s.slug)) + '</code></span>');
  return pills.join('');
}
```

---

## Что проверить после вставки

1. Открываешь `index.php` → у каждой карточки появилась кнопка **Настройки**.
2. Кликаешь → открывается `/local/sitebuilder/settings.php?siteId=...`
3. Если нет прав EDITOR — settings.php откроется, но при сохранении получишь FORBIDDEN (это ожидаемо).

---

Если хочешь — я могу сразу **встроить “Настройки” не в новую вкладку**, а в тот же интерфейс (как “Страницы”/“Доступы” модалкой). Но для серьёзного проекта текущий вариант (отдельная страница) — более устойчивый и расширяемый.
