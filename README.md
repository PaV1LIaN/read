Ага, по скрину сразу понятно, почему “как будто не выбирается”.

У меня в той версии **selection реально сбрасывается при каждом перерисовывании**:

* ты кликаешь по карточке → я ставлю `picked_parent = id`
* **сразу делаю rerender**
* а `renderListInner()` снова рисует `<input id="picked_parent" value="...">` **со старым `currentParentId`**
* в итоге визуально ничего не меняется и выбор “пропадает”.

То есть обработчик клика может срабатывать, но мы сами же затираем выбор.

## ✅ Исправление (1 переменная selectedParentId)

Заменяешь свою текущую `openSetParentDialog(...)` на эту версию (она такая же, но с `selectedParentId`, который обновляется при клике и используется в рендере):

```js
function openSetParentDialog(siteId, pageId, pagesCache, reload) {
  const pages = Array.isArray(pagesCache) ? pagesCache : [];
  const current = pages.find(x => parseInt(x.id,10) === parseInt(pageId,10));
  const currentParentId = parseInt(current?.parentId || 0, 10) || 0;

  // ВАЖНО: выбранный родитель должен быть изменяемым, а не константой
  let selectedParentId = currentParentId;

  const { roots } = buildTree(pages);
  const flat = [];
  const walk = (arr, depth) => {
    arr.forEach(n => {
      flat.push({ id: n.id, title: n.title||'', slug:n.slug||'', depth });
      walk(n.children||[], depth+1);
    });
  };
  walk(roots, 0);

  const renderListInner = (q) => {
    const query = (q||'').trim().toLowerCase();

    const items = flat.filter(x => {
      if (parseInt(x.id,10) === parseInt(pageId,10)) return false; // нельзя выбрать себя
      if (!query) return true;
      return (x.title||'').toLowerCase().includes(query)
        || (x.slug||'').toLowerCase().includes(query)
        || String(x.id).includes(query);
    });

    const rows = items.map(x => {
      const pad = 12 + x.depth * 16;
      const active = (parseInt(x.id,10) === selectedParentId)
        ? 'background:#eff6ff;border-color:#bfdbfe;'
        : '';
      return `
        <div class="secCard" data-parent-pick="${x.id}"
             style="cursor:pointer; padding-left:${pad}px; ${active}">
          <div class="secTitle">#${x.id} ${BX.util.htmlspecialchars(x.title)}</div>
          <div class="secMeta">${BX.util.htmlspecialchars(x.slug)}</div>
        </div>
      `;
    }).join('');

    const activeRoot = selectedParentId === 0 ? 'background:#eff6ff;border-color:#bfdbfe;' : '';

    return `
      <div class="secSearch">
        <input id="par_q" class="input" placeholder="Поиск родителя..." value="${BX.util.htmlspecialchars(q||'')}">
      </div>

      <div class="secGrid" style="grid-template-columns: 1fr; margin-top:10px;">
        <div class="secCard" data-parent-pick="0" style="cursor:pointer; ${activeRoot}">
          <div class="secTitle">— Корень —</div>
          <div class="secMeta">Сделать страницу верхнего уровня</div>
        </div>

        ${rows || '<div class="muted">Ничего не найдено</div>'}
      </div>

      <div class="hint" style="margin-top:10px;">Кликни по карточке родителя. Потом нажми OK.</div>
    `;
  };

  BX.UI.Dialogs.MessageBox.show({
    title: 'Вложить страницу #' + pageId,
    message: `<div id="sb_parent_picker_root">${renderListInner('')}</div>`,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      const parentId = parseInt(selectedParentId || 0, 10) || 0;

      api('page.setParent', { id: pageId, parentId }).then(r=>{
        if (!r || r.ok !== true) { notify('Не удалось изменить parent'); return; }
        notify('Готово');
        mb.close();
        reload();
      }).catch(()=>notify('Ошибка page.setParent'));
    }
  });

  setTimeout(() => {
    const root = document.getElementById('sb_parent_picker_root');
    if (!root) return;

    const bind = () => {
      const q = document.getElementById('par_q');
      if (q) {
        q.oninput = () => {
          root.innerHTML = renderListInner(q.value);
          bind();
        };
      }

      root.querySelectorAll('[data-parent-pick]').forEach(el => {
        el.onclick = () => {
          const id = parseInt(el.getAttribute('data-parent-pick')||'0',10) || 0;
          selectedParentId = id; // ✅ сохраняем выбор
          const curQ = document.getElementById('par_q')?.value || '';
          root.innerHTML = renderListInner(curQ); // ✅ перерисовка уже с новым selectedParentId
          bind();
        };
      });
    };

    bind();
  }, 0);
}
```

### Что поменяется после фикса

* при клике карточка будет **подсвечиваться** (как выбранная)
* и при OK будет уходить **правильный parentId** в `page.setParent`.

Если после этого подсветка есть, но parent не меняется — тогда уже посмотрим **ответ API** (403/422) и что приходит в `page.setParent`.
