Понял. По твоему коду **почти наверняка bind() вообще не вешается**, потому что ты ищешь DOM MessageBox вот так:

```js
document.querySelector('.bx-ui-dialogs-messagebox')
  .querySelector('.bx-ui-dialogs-messagebox-content')
```

А в реальной разметке Bitrix UI эти классы **часто другие / вложенность другая**, поэтому `root === null`, и **кнопки внутри каталога остаются “мертвыми”** (потому что им не назначили onclick).

✅ Самый надёжный фикс: **не искать внутренности MessageBox по классам**, а **вставить свой контейнер с id** и потом брать его через `getElementById`. Плюс сделать **делегирование кликов**, чтобы после `rerender()` ничего не “отваливалось”.

Ниже — готовая замена функции `openSectionsLibrary()` (только её). Скопируй и **полностью замени** свою текущую `openSectionsLibrary()` на эту.

---

## ✅ Замени `openSectionsLibrary()` на это

```js
async function openSectionsLibrary() {
  let res;
  try { res = await api('template.list', {}); }
  catch (e) { notify('Ошибка template.list'); return; }

  if (!res || res.ok !== true) { notify('Не удалось получить шаблоны'); return; }

  let templates = res.templates || [];
  if (!templates.length) { notify('Шаблонов нет. Сначала сохрани страницу как шаблон.'); return; }

  const containerId = 'sb_sections_root_' + Date.now();

  const render = (q) => {
    const query = (q || '').trim().toLowerCase();
    const filtered = templates.filter(t => ((t.name || '') + '').toLowerCase().includes(query));

    const cards = filtered.map(t => {
      const blocksCount = Array.isArray(t.blocks) ? t.blocks.length : 0;
      const createdAt = (t.createdAt || '').replace('T',' ').replace('Z','');

      return `
        <div class="secCard">
          <div class="secTitle">${BX.util.htmlspecialchars(t.name || ('Template #' + t.id))}</div>
          <div class="secMeta">id: ${t.id} • блоков: ${blocksCount} • создан: ${BX.util.htmlspecialchars(createdAt)}</div>

          <div class="secBtns">
            <button class="ui-btn ui-btn-primary ui-btn-xs" data-tpl-apply="${t.id}" data-mode="append">Вставить</button>
            <button class="ui-btn ui-btn-light ui-btn-xs" data-tpl-apply="${t.id}" data-mode="replace">Заменить</button>
            <button class="ui-btn ui-btn-light ui-btn-xs" data-tpl-rename="${t.id}">Переименовать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-tpl-delete="${t.id}">Удалить</button>
          </div>
        </div>
      `;
    }).join('');

    return `
      <div id="${containerId}">
        <div class="secSearch">
          <input id="${containerId}_q" class="input" placeholder="Поиск шаблонов..." value="${BX.util.htmlspecialchars(q || '')}">
        </div>
        <div class="secGrid">${cards || '<div class="muted">Ничего не найдено</div>'}</div>
      </div>
    `;
  };

  BX.UI.Dialogs.MessageBox.show({
    title: 'Каталог секций',
    message: render(''),
    buttons: BX.UI.Dialogs.MessageBoxButtons.CANCEL,
    onCancel: function (mb) { mb.close(); }
  });

  setTimeout(() => {
    const root = document.getElementById(containerId);
    if (!root) {
      notify('Каталог секций: не найден контейнер (проверь messagebox DOM)');
      return;
    }

    const rerender = (q) => {
      root.outerHTML = render(q);
      const newRoot = document.getElementById(containerId);
      if (!newRoot) return;

      // заново вешаем обработчики на новый корень
      bind(newRoot);
    };

    const bind = (r) => {
      // поиск
      const q = document.getElementById(containerId + '_q');
      if (q) q.oninput = () => rerender(q.value);

      // делегирование кликов по кнопкам в карточках
      r.onclick = async (e) => {
        const applyBtn = e.target.closest('[data-tpl-apply]');
        if (applyBtn) {
          const tplId = parseInt(applyBtn.getAttribute('data-tpl-apply'), 10);
          const mode = applyBtn.getAttribute('data-mode') || 'append';
          try {
            const r2 = await api('template.applyToPage', { siteId, pageId, templateId: tplId, mode });
            if (!r2 || r2.ok !== true) { notify('Не удалось применить'); return; }
            notify('Готово: добавлено блоков ' + (r2.added || 0));
            loadBlocks();
          } catch (err) {
            notify('Ошибка apply');
          }
          return;
        }

        const renameBtn = e.target.closest('[data-tpl-rename]');
        if (renameBtn) {
          const tplId = parseInt(renameBtn.getAttribute('data-tpl-rename'), 10);
          const cur = templates.find(x => parseInt(x.id, 10) === tplId);

          BX.UI.Dialogs.MessageBox.show({
            title: 'Переименовать',
            message: `<div class="field"><label>Название</label><input id="rn_name" class="input" value="${BX.util.htmlspecialchars(cur?.name || '')}"></div>`,
            buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
            onOk: function (mb2) {
              const name = (document.getElementById('rn_name')?.value || '').trim();
              if (!name) { notify('Введите название'); return; }

              api('template.rename', { id: tplId, name }).then(r3 => {
                if (!r3 || r3.ok !== true) { notify('Не удалось переименовать'); return; }
                notify('Переименовано');
                const idx = templates.findIndex(x => parseInt(x.id, 10) === tplId);
                if (idx >= 0) templates[idx].name = name;
                mb2.close();
                const qv = document.getElementById(containerId + '_q')?.value || '';
                rerender(qv);
              }).catch(() => notify('Ошибка template.rename'));
            }
          });
          return;
        }

        const delBtn = e.target.closest('[data-tpl-delete]');
        if (delBtn) {
          const tplId = parseInt(delBtn.getAttribute('data-tpl-delete'), 10);

          BX.UI.Dialogs.MessageBox.show({
            title: 'Удалить шаблон?',
            message: 'Удалить навсегда?',
            buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
            onOk: function (mb2) {
              api('template.delete', { id: tplId }).then(r4 => {
                if (!r4 || r4.ok !== true) { notify('Не удалось удалить'); return; }
                notify('Удалено');
                templates = templates.filter(x => parseInt(x.id, 10) !== tplId);
                mb2.close();
                const qv = document.getElementById(containerId + '_q')?.value || '';
                rerender(qv);
              }).catch(() => notify('Ошибка template.delete'));
            }
          });
          return;
        }
      };
    };

    bind(root);
  }, 0);
}
```

---

## Почему так заработает

* Мы не зависим от внутренних классов Bitrix MessageBox.
* Всегда точно находим контент по `id`.
* После `rerender()` события не теряются, потому что:

  * поиск перевешивается,
  * клики обрабатываются **делегированно** через `root.onclick`.

---

## Проверь ещё один момент

Внизу у тебя уже есть:

```js
btnSections.addEventListener('click', openSectionsLibrary);
```

Это оставь.

---

Если после замены кнопки всё равно не реагируют — скажи, появляется ли в консоли сообщение:
`Каталог секций: не найден контейнер...`
(если да — значит Bitrix экранирует HTML в `message:`; тогда будем использовать `message: ''` и вставлять через `onShow`/PopupWindow контейнер — тоже решаемо).
