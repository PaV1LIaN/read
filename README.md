Понял. Это классическая ситуация: **кнопки/селект “Вложить” рисуются внутри MessageBox динамически**, а обработчик либо **не навешан**, либо **навешан на элементы, которых ещё нет**, либо **не делается API-вызов `page.setParent`** (у тебя он есть в `api.php`, но UI в `index.php` из того что ты прислал — его вообще не вызывает).

Сделаем правильно и “железобетонно”: **через делегирование событий `document.addEventListener('click', ...)`**, как у тебя уже сделано для удаления страницы.

Ниже — готовый патч для `index.php` (JS-часть).

---

## 1) Добавь кнопки “Вложить” в список страниц

В функции `renderPages(container, pages)` найди где ты рисуешь действия:

```js
<td ...>
  <a ...>Редактор</a>
  <a ...>Открыть</a>
  <button ... data-delete-page-id="...">Удалить</button>
</td>
```

Замени на вот так (добавим “Вложить” и “Сделать корневой”):

```js
<td style="padding:8px;border-bottom:1px solid #eee; white-space:nowrap;">
  <a class="ui-btn ui-btn-primary ui-btn-xs"
     href="/local/sitebuilder/editor.php?siteId=${p.siteId}&pageId=${p.id}"
     target="_blank">Редактор</a>

  <a class="ui-btn ui-btn-light ui-btn-xs"
     href="/local/sitebuilder/view.php?siteId=${p.siteId}&pageId=${p.id}"
     target="_blank">Открыть</a>

  <button class="ui-btn ui-btn-light ui-btn-xs"
          data-page-nest-id="${p.id}"
          data-page-nest-site-id="${p.siteId}">Вложить</button>

  <button class="ui-btn ui-btn-light ui-btn-xs"
          data-page-root-id="${p.id}"
          data-page-root-site-id="${p.siteId}">Сделать корневой</button>

  <button class="ui-btn ui-btn-danger ui-btn-xs"
          data-delete-page-id="${p.id}"
          data-delete-page-site-id="${p.siteId}">Удалить</button>
</td>
```

> Примечание: у тебя в `pages.json` есть `parentId`, но UI его не показывает. Позже добавим колонку “Родитель”, но сейчас главное — чтобы “вложить” заработало.

---

## 2) Добавь обработчики кликов “Вложить” и “Сделать корневой”

Внизу `index.php` у тебя уже есть:

```js
document.addEventListener('click', function (e) {
  ... delete site ...
  ... pages dialog ...
  ... access dialog ...
  ... delete page ...
  ... delete access ...
});
```

ВНУТРИ этого обработчика (лучше после блока удаления страницы, но до удаления access — не важно) вставь:

```js
// ---------- PAGE NESTING (set parent) ----------
const nestBtn = e.target.closest('[data-page-nest-id]');
if (nestBtn) {
  const pageId = parseInt(nestBtn.getAttribute('data-page-nest-id'), 10);
  const siteId = parseInt(nestBtn.getAttribute('data-page-nest-site-id'), 10);
  if (!pageId || !siteId) return;

  // возьмём список страниц, чтобы выбрать родителя
  api('page.list', { siteId }).then(res => {
    if (!res || res.ok !== true) {
      BX.UI.Notification.Center.notify({ content: 'Не удалось загрузить страницы' });
      return;
    }
    const pages = res.pages || [];
    const options = pages
      .filter(x => parseInt(x.id, 10) !== pageId) // нельзя вложить в саму себя
      .map(x => `<option value="${x.id}">${BX.util.htmlspecialchars(x.title)} (id ${x.id})</option>`)
      .join('');

    BX.UI.Dialogs.MessageBox.show({
      title: 'Вложить страницу #' + pageId,
      message: `
        <div class="muted" style="margin-bottom:8px;">Выбери родительскую страницу:</div>
        <select id="sb_parent_id" style="width:100%; padding:8px; border:1px solid #d0d7de; border-radius:8px;">
          <option value="0">— Корень —</option>
          ${options}
        </select>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb){
        const parentId = parseInt(document.getElementById('sb_parent_id')?.value || '0', 10);

        api('page.setParent', { id: pageId, parentId }).then(r => {
          if (!r || r.ok !== true) {
            BX.UI.Notification.Center.notify({ content: 'Не удалось изменить вложенность (нужен EDITOR+)' });
            return;
          }
          BX.UI.Notification.Center.notify({ content: 'Вложенность сохранена' });
          mb.close();

          // если открыто окно страниц — перерисуем его
          const pagesBox = document.getElementById('pagesBox');
          if (pagesBox) loadPages(siteId, pagesBox);
          loadSites();
        }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка page.setParent' }));
      }
    });
  });

  return;
}

const rootBtn = e.target.closest('[data-page-root-id]');
if (rootBtn) {
  const pageId = parseInt(rootBtn.getAttribute('data-page-root-id'), 10);
  const siteId = parseInt(rootBtn.getAttribute('data-page-root-site-id'), 10);
  if (!pageId || !siteId) return;

  api('page.setParent', { id: pageId, parentId: 0 }).then(r => {
    if (!r || r.ok !== true) {
      BX.UI.Notification.Center.notify({ content: 'Не удалось сделать корневой (нужен EDITOR+)' });
      return;
    }
    BX.UI.Notification.Center.notify({ content: 'Страница стала корневой' });

    const pagesBox = document.getElementById('pagesBox');
    if (pagesBox) loadPages(siteId, pagesBox);
    loadSites();
  }).catch(() => BX.UI.Notification.Center.notify({ content: 'Ошибка page.setParent' }));

  return;
}
```

---

## Почему “ничего не происходит” у тебя сейчас

Потому что:

* либо кнопка “Вложить” есть, но **у неё нет обработчика**,
* либо обработчик навешан напрямую на кнопку **до того как она появилась**,
* либо обработчик есть, но **не делает `api('page.setParent', ...)`**,
* либо `api('page.setParent')` вызывается, но **ты не обновляешь список страниц**, и кажется что “ничего не произошло”.

Код выше закрывает все эти варианты.

---

Если после вставки всё равно “тишина” — в консоли будет ошибка (обычно `page.setParent UNKNOWN_ACTION` или 403). Скинь 1 строку из Network (ответ api.php) — и я точечно скажу, что именно блокирует (роль, параметры или action).
