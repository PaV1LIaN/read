Ок, делаем **п.5 (menu.php: кнопка “Сделать верхним”)**.
Я дам **точно куда вставлять** и **что заменить**, потому что у тебя сейчас в `menu.php` уже есть заготовка, но там есть 2 ошибки, из-за которых “верхнее меню” нормально не подтянется:

* в `loadAll()` ты пишешь `api('site.get')`, но переменная у тебя названа `pagesRes/menusRes`, а возвращаешь `siteRes` (которого нет)
* `render()` принимает `{menus, pages, topMenuId}`, а ты передаёшь `render(data)` где `data.site.topMenuId` лежит внутри `site`.

Ниже — **минимальные правки** в `menu.php`:

---

## 1) loadAll — замени целиком функцию

Найди в `menu.php`:

```js
async function loadAll() {
  const [menusRes, pagesRes] = await Promise.all([
    api('menu.list'),
    api('page.list'),
    api('site.get')
  ]);

  if (!menusRes || menusRes.ok !== true) throw new Error('menu.list failed');
  if (!pagesRes || pagesRes.ok !== true) throw new Error('page.list failed');

  return { menus: menusRes.menus||[], pages: pagesRes.pages||[], site: siteRes.site||{} };
}
```

И **замени полностью** на:

```js
async function loadAll() {
  const [menusRes, pagesRes, siteRes] = await Promise.all([
    api('menu.list'),
    api('page.list'),
    api('site.get', { siteId }) // можно и без siteId, но так надёжнее
  ]);

  if (!menusRes || menusRes.ok !== true) throw new Error('menu.list failed');
  if (!pagesRes || pagesRes.ok !== true) throw new Error('page.list failed');
  if (!siteRes || siteRes.ok !== true) throw new Error('site.get failed');

  return {
    menus: menusRes.menus || [],
    pages: pagesRes.pages || [],
    site: siteRes.site || {}
  };
}
```

---

## 2) render — поправь сигнатуру и использование topMenuId

Найди:

```js
function render({menus, pages, topMenuId}) {
```

И замени на:

```js
function render({ menus, pages, site }) {
  const topMenuId = parseInt(site?.topMenuId || 0, 10) || 0;
```

> Внутри `render()` дальше код с `topMenuId` у тебя уже правильный, он будет работать.

---

## 3) Кнопка “Сделать верхним” — HTML уже вставлен правильно

Вот этот кусок у тебя **правильный** и должен быть **внутри** `menus.map(m => ...)` в `render()`:

```js
${parseInt(m.id,10) === (topMenuId||0)
  ? `<span class="muted small" style="padding:6px 8px;">Верхнее меню</span>`
  : `<button class="ui-btn ui-btn-success ui-btn-xs" data-menu-set-top="${m.id}">Сделать верхним</button>`
}
```

Если ты всё же сомневаешься “куда” — он должен стоять **в шапке карточки меню**, рядом с “Переименовать” и “+ Пункт”.

---

## 4) Обработчик клика — у тебя он тоже по месту, но проверь где стоит

Этот блок должен быть **внутри**:

```js
document.addEventListener('click', async function (e) {
  ...
});
```

и **после** остальных обработчиков (или до — неважно), но точно **до закрывающей скобки** этого listener.

Вот корректный обработчик (можешь заменить свой `const st...` блок полностью):

```js
const st = e.target.closest('[data-menu-set-top]');
if (st) {
  const menuId = parseInt(st.getAttribute('data-menu-set-top'), 10);
  api('menu.setTop', { menuId }).then(res => {
    if (!res || res.ok !== true) {
      BX.UI.Notification.Center.notify({ content: 'Не удалось назначить верхнее меню (нужен EDITOR+)' });
      return;
    }
    BX.UI.Notification.Center.notify({ content: 'Верхнее меню назначено' });
    refresh();
  }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка menu.setTop' }));
  return;
}
```

---

## Важно: без API это не заработает

`menu.php` вызывает:

* `api('site.get')`
* `api('menu.setTop')`

Значит в `api.php` должны быть обработчики:

* `site.get` → вернуть объект сайта (хотя бы `topMenuId`)
* `menu.setTop` → сохранить `topMenuId` в `sites.json`

Если хочешь — я прямо сейчас дам **готовые два action’а для api.php** (это следующий шаг), но ты спросил именно “куда и что вставлять в menu.php” — выше всё по местам.

Скажи только: `sites.json` у тебя уже содержит `topMenuId` или пока нет? Если нет — `menu.setTop` просто добавит поле при записи.
