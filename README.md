Понял. По первому пункту: **“вложить…” показывает только корень**, значит у тебя в диалог “родитель” не попадает список страниц (кеш пустой). В моём коде он лежал в `pagesContainer.__sb_pages`, но это иногда реально “ломается” (например, если кликнуть быстро до загрузки, или если контейнер пересоздался/перерендерился).

Сделаем **надёжно** и заодно **перерисуем дизайн**, чтобы было интуитивно:

1. страницы **кешируем в переменную** `pagesCache` внутри `openPagesDialog`, а не в DOM property
2. “Вложить…” откроет **красивый пикер родителя**: поиск + дерево с отступами, кликом выбираешь родителя
3. внешний вид списка страниц: более “деревянный” и читабельный, кнопки компактные, заголовок/slug/мета разнесены

Ниже — **что именно заменить** в `index.php` (внутри JS, в функции `openPagesDialog` и рядом).

---

## 1) Заменить CSS для дерева (в `<style>`)

Найди блок `/* pages tree */ ...` и замени на это:

```css
/* pages tree (better UI) */
.tree { margin-top: 12px; }
.node { border:1px solid #eef0f2; border-radius:14px; padding:12px; background:#fff; margin-top:10px; }
.nodeHead { display:flex; gap:12px; align-items:flex-start; justify-content:space-between; flex-wrap:wrap; }
.nodeLeft { display:flex; gap:10px; align-items:flex-start; }
.nodeIcon {
  width:22px; height:22px; border-radius:8px; background:#f3f4f6;
  display:flex; align-items:center; justify-content:center; color:#6a737f; font-weight:700;
  flex:0 0 auto;
}
.nodeMain { min-width: 240px; }
.nodeTitleLine { display:flex; gap:8px; align-items:center; flex-wrap:wrap; }
.nodeTitleLine b { font-size:14px; }
.nodeSlug { font-size:12px; background:#f3f4f6; padding:2px 8px; border-radius:999px; color:#374151; }
.nodeMeta { margin-top:4px; font-size:12px; color:#6a737f; display:flex; gap:10px; flex-wrap:wrap; }
.nodeBtns { display:flex; gap:6px; flex-wrap:wrap; align-items:center; }
.children { margin-left:18px; border-left:2px dashed #e5e7ea; padding-left:12px; margin-top:10px; }
.hint { font-size:12px; color:#6a737f; margin-top:8px; }

.btnTiny { padding:0 8px; height:26px; line-height:26px; }
```

---

## 2) Заменить renderPagesTree + openPagesDialog на “кеш + нормальный вид”

Найди в JS функции:

* `renderPagesTree(...)`
* `openSetParentDialog(...)`
* `openPagesDialog(...)`

и замени **полностью** на версии ниже (они совместимы с твоими API: `page.list`, `page.move`, `page.setParent`, `page.updateMeta`, `page.delete`).

### ✅ Вставь этот блок вместо старых функций

```js
function buildTree(pages) {
  const byId = {};
  pages.forEach(p => { byId[p.id] = Object.assign({ children: [] }, p); });

  const roots = [];
  pages.forEach(p => {
    const pid = parseInt(p.parentId || 0, 10) || 0;
    if (pid && byId[pid]) byId[pid].children.push(byId[p.id]);
    else roots.push(byId[p.id]);
  });

  const sortFn = (a,b) => (parseInt(a.sort||500,10) - parseInt(b.sort||500,10)) || (a.id - b.id);
  const sortRec = (arr) => { arr.sort(sortFn); arr.forEach(x => sortRec(x.children)); };
  sortRec(roots);

  return { roots, byId };
}

function renderPagesTree(container, siteId, pages, q) {
  const query = (q || '').trim().toLowerCase();
  const matches = (p) => {
    if (!query) return true;
    const t = (p.title||'').toLowerCase();
    const s = (p.slug||'').toLowerCase();
    return t.includes(query) || s.includes(query) || String(p.id).includes(query);
  };

  const { roots } = buildTree(pages);

  const renderNode = (node) => {
    const kidsHtml = (node.children || [])
      .map(renderNode)
      .filter(Boolean)
      .join('');

    const selfMatch = matches(node);
    const hasVisibleKids = kidsHtml !== '';
    if (!selfMatch && !hasVisibleKids) return '';

    const pid = parseInt(node.parentId||0,10)||0;
    const parentLabel = pid ? `parent: #${pid}` : 'root';

    return `
      <div class="node">
        <div class="nodeHead">
          <div class="nodeLeft">
            <div class="nodeIcon">≡</div>
            <div class="nodeMain">
              <div class="nodeTitleLine">
                <b>#${node.id} ${BX.util.htmlspecialchars(node.title || '')}</b>
                <span class="nodeSlug">${BX.util.htmlspecialchars(node.slug || '')}</span>
              </div>
              <div class="nodeMeta">
                <span>sort: ${parseInt(node.sort||500,10)}</span>
                <span>${parentLabel}</span>
              </div>
            </div>
          </div>

          <div class="nodeBtns">
            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="up">↑</button>
            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-move="${node.id}" data-dir="down">↓</button>

            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-parent="${node.id}">Вложить…</button>
            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-root="${node.id}">В корень</button>

            <button class="ui-btn ui-btn-light ui-btn-xs btnTiny" data-page-rename="${node.id}">Имя/slug</button>

            <a class="ui-btn ui-btn-primary ui-btn-xs btnTiny"
               href="/local/sitebuilder/editor.php?siteId=${siteId}&pageId=${node.id}"
               target="_blank">Редактор</a>

            <a class="ui-btn ui-btn-light ui-btn-xs btnTiny"
               href="/local/sitebuilder/view.php?siteId=${siteId}&pageId=${node.id}"
               target="_blank">Открыть</a>

            <button class="ui-btn ui-btn-danger ui-btn-xs btnTiny" data-page-delete="${node.id}">Удалить</button>
          </div>
        </div>

        ${hasVisibleKids ? `<div class="children">${kidsHtml}</div>` : ''}
      </div>
    `;
  };

  const html = roots.map(renderNode).filter(Boolean).join('');
  container.innerHTML = html || '<div class="muted">Страниц пока нет.</div>';
}

function openRenamePageDialog(siteId, pageId, pagesCache, reload) {
  const cur = (pagesCache || []).find(x => parseInt(x.id,10) === parseInt(pageId,10));
  const curTitle = cur?.title || '';
  const curSlug  = cur?.slug || '';

  BX.UI.Dialogs.MessageBox.show({
    title: 'Имя / slug для #' + pageId,
    message: `
      <div style="display:flex; flex-direction:column; gap:10px;">
        <div class="field">
          <label>Заголовок</label>
          <input id="rn_title" class="input" value="${BX.util.htmlspecialchars(curTitle)}" />
        </div>
        <div class="field">
          <label>Slug (можно пусто — пересчитаем)</label>
          <input id="rn_slug" class="input" value="${BX.util.htmlspecialchars(curSlug)}" />
        </div>
        <div class="hint">Slug должен быть уникален в пределах сайта — если совпадёт, добавим суффикс.</div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      const title = (document.getElementById('rn_title')?.value || '').trim();
      const slug = (document.getElementById('rn_slug')?.value || '').trim();
      if (!title) { notify('Заголовок не может быть пустым'); return; }

      api('page.updateMeta', { id: pageId, title, slug }).then(r => {
        if (!r || r.ok !== true) { notify('Не удалось сохранить'); return; }
        notify('Сохранено');
        mb.close();
        reload();
      }).catch(()=>notify('Ошибка page.updateMeta'));
    }
  });
}

function openSetParentDialog(siteId, pageId, pagesCache, reload) {
  const pages = Array.isArray(pagesCache) ? pagesCache : [];
  const current = pages.find(x => parseInt(x.id,10) === parseInt(pageId,10));
  const currentParentId = parseInt(current?.parentId || 0, 10) || 0;

  // строим дерево и плоский список с depth (чтобы красиво рисовать)
  const { roots } = buildTree(pages);
  const flat = [];
  const walk = (arr, depth) => {
    arr.forEach(n => {
      flat.push({ id: n.id, title: n.title||'', slug:n.slug||'', depth });
      walk(n.children||[], depth+1);
    });
  };
  walk(roots, 0);

  const renderList = (q) => {
    const query = (q||'').trim().toLowerCase();
    const items = flat.filter(x => {
      if (x.id === pageId) return false; // нельзя выбрать себя
      if (!query) return true;
      return (x.title||'').toLowerCase().includes(query)
        || (x.slug||'').toLowerCase().includes(query)
        || String(x.id).includes(query);
    });

    const rows = items.map(x => {
      const pad = 8 + x.depth * 16;
      const active = (parseInt(x.id,10) === currentParentId) ? 'style="background:#eff6ff;border-color:#bfdbfe;"' : '';
      return `
        <div class="secCard" ${active} data-parent-pick="${x.id}" style="cursor:pointer; padding-left:${pad}px;">
          <div class="secTitle">#${x.id} ${BX.util.htmlspecialchars(x.title)}</div>
          <div class="secMeta">${BX.util.htmlspecialchars(x.slug)}</div>
        </div>
      `;
    }).join('');

    return `
      <div>
        <div class="secSearch">
          <input id="par_q" class="input" placeholder="Поиск родителя..." value="${BX.util.htmlspecialchars(q||'')}">
        </div>

        <div class="secGrid" style="grid-template-columns: 1fr; margin-top:10px;">
          <div class="secCard" data-parent-pick="0" style="cursor:pointer; ${currentParentId===0?'background:#eff6ff;border-color:#bfdbfe;':''}">
            <div class="secTitle">— Корень —</div>
            <div class="secMeta">Сделать страницу верхнего уровня</div>
          </div>

          ${rows || '<div class="muted">Ничего не найдено</div>'}
        </div>

        <input type="hidden" id="picked_parent" value="${currentParentId}">
        <div class="hint" style="margin-top:10px;">Кликни по карточке родителя. Потом нажми OK.</div>
      </div>
    `;
  };

  BX.UI.Dialogs.MessageBox.show({
    title: 'Вложить страницу #' + pageId,
    message: renderList(''),
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      const parentId = parseInt(document.getElementById('picked_parent')?.value || '0', 10) || 0;
      api('page.setParent', { id: pageId, parentId }).then(r=>{
        if (!r || r.ok !== true) { notify('Не удалось изменить parent'); return; }
        notify('Готово');
        mb.close();
        reload();
      }).catch(()=>notify('Ошибка page.setParent'));
    }
  });

  setTimeout(() => {
    const mb = document.querySelector('.bx-ui-dialogs-messagebox');
    const root = mb ? mb.querySelector('.bx-ui-dialogs-messagebox-content') : null;
    if (!root) return;

    const rerender = (q) => { root.innerHTML = renderList(q); bind(); };

    const bind = () => {
      const q = document.getElementById('par_q');
      if (q) q.oninput = () => rerender(q.value);

      root.querySelectorAll('[data-parent-pick]').forEach(el => {
        el.onclick = () => {
          const id = parseInt(el.getAttribute('data-parent-pick')||'0',10)||0;
          const hid = document.getElementById('picked_parent');
          if (hid) hid.value = String(id);
          // подсветка (простой rerender чтобы подсветилось)
          const curQ = document.getElementById('par_q')?.value || '';
          rerender(curQ);
        };
      });
    };

    bind();
  }, 0);
}

async function openPagesDialog(siteId, siteName) {
  let pagesCache = [];

  const html = `
    <div>
      <div class="searchRow">
        <div>
          <button class="ui-btn ui-btn-primary ui-btn-xs" id="btnCreatePage">Создать страницу</button>
        </div>
        <div class="field">
          <label>Поиск</label>
          <input id="pg_q" class="input" placeholder="title / slug / id..." />
        </div>
      </div>

      <div class="hint">
        Дерево: <code>parentId</code>. Порядок: <code>sort</code> (стрелки ↑/↓ меняют порядок среди “соседей”).
      </div>

      <div id="pagesBox" class="tree"></div>
    </div>
  `;

  BX.UI.Dialogs.MessageBox.show({
    title: 'Страницы сайта: ' + BX.util.htmlspecialchars(siteName),
    message: html,
    buttons: BX.UI.Dialogs.MessageBoxButtons.CLOSE
  });

  const loadAndRender = async () => {
    const container = document.getElementById('pagesBox');
    const q = document.getElementById('pg_q')?.value || '';
    if (!container) return;

    try {
      const res = await api('page.list', { siteId });
      if (!res || res.ok !== true) { notify('Не удалось загрузить страницы (возможно нет прав)'); return; }
      pagesCache = res.pages || [];
      renderPagesTree(container, siteId, pagesCache, q);
    } catch (e) {
      notify('Ошибка page.list');
    }
  };

  setTimeout(() => {
    const container = document.getElementById('pagesBox');
    const q = document.getElementById('pg_q');
    const btn = document.getElementById('btnCreatePage');

    if (btn) btn.onclick = () => openCreatePageDialog(siteId, container);
    if (q) q.oninput = () => loadAndRender();

    if (container) {
      container.addEventListener('click', function(e){
        const mv = e.target.closest('[data-page-move]');
        if (mv) {
          const id = parseInt(mv.getAttribute('data-page-move'),10);
          const dir = mv.getAttribute('data-dir');
          api('page.move', { id, dir }).then(r=>{
            if(!r || r.ok!==true){ notify('Не удалось переместить'); return; }
            loadAndRender();
          }).catch(()=>notify('Ошибка page.move'));
          return;
        }

        const rn = e.target.closest('[data-page-rename]');
        if (rn) {
          const id = parseInt(rn.getAttribute('data-page-rename'),10);
          openRenamePageDialog(siteId, id, pagesCache, loadAndRender);
          return;
        }

        const pr = e.target.closest('[data-page-parent]');
        if (pr) {
          const id = parseInt(pr.getAttribute('data-page-parent'),10);
          openSetParentDialog(siteId, id, pagesCache, loadAndRender);
          return;
        }

        const rt = e.target.closest('[data-page-root]');
        if (rt) {
          const id = parseInt(rt.getAttribute('data-page-root'),10);
          api('page.setParent', { id, parentId: 0 }).then(r=>{
            if(!r || r.ok!==true){ notify('Не удалось'); return; }
            loadAndRender();
          }).catch(()=>notify('Ошибка page.setParent'));
          return;
        }

        const del = e.target.closest('[data-page-delete]');
        if (del) {
          const id = parseInt(del.getAttribute('data-page-delete'),10);
          BX.UI.Dialogs.MessageBox.show({
            title: 'Удалить страницу #' + id + '?',
            message: 'Нужны права EDITOR/ADMIN/OWNER. Продолжить?',
            buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
            onOk: function(mb){
              api('page.delete', { id }).then(r=>{
                if(!r || r.ok!==true){ notify('Не удалось удалить (возможно нет прав)'); return; }
                notify('Страница удалена');
                mb.close();
                loadAndRender();
              }).catch(()=>notify('Ошибка page.delete'));
            }
          });
          return;
        }
      });
    }

    loadAndRender();
  }, 0);
}
```

---

## Почему “только корень” было раньше

Почти наверняка из-за того, что список страниц в “родитель” брался из `pagesContainer.__sb_pages`, а он у тебя был пустой/не успевал заполниться.
Теперь мы используем **`pagesCache`**, который заполняется после `page.list`, и передаём его напрямую в диалоги — так будет стабильно.

---

## Что проверить сейчас

1. Открой “Страницы” → нажми **Вложить…**
   Должно открыться окно с:

   * **Корень**
   * список страниц (с отступами как дерево)
   * поиск
2. Выбери родителя кликом → OK → дерево обновится

---

Если хочешь ещё “прям совсем” UX:

* добавить **иконки** вместо текста кнопок
* сделать **контекстное меню** (⋯) вместо пачки кнопок
* добавить кнопку **“+ дочернюю страницу”** прямо у узла

Скажи, какой стиль тебе ближе: “минимализм (контекст-меню)” или “все кнопки видны”?
