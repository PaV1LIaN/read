Ошибка понятная: `addCardsBlock()` вызывается, **но функция `openCardsBuilderDialog` не попала в область видимости** (её нет в файле или она объявлена *ниже* так, что код до неё не дошёл из-за синтаксической ошибки/раннего выхода).

Сделаем так, чтобы ты **точно** заработал за 2 минуты: я дам **готовый минимальный рабочий блок**, который ты просто вставишь в `editor.php` **в одно место** — и всё.

---

# ✅ Шаг 1. Вставь `openCardsBuilderDialog` (и хелперы) ВЫШЕ `function addCardsBlock()`

Найди в `editor.php` место, где у тебя объявлены функции `addCardBlock() / editCardBlock()` или вообще любой “диалоговый” блок — **и вставь этот код ПЕРЕД `function addCardsBlock()`**.

```js
// ===== CARDS BUILDER (визуальный конструктор) =====

function cardsNormalizeItem(it) {
  const x = (it && typeof it === 'object') ? it : {};
  return {
    title: (x.title || '').toString(),
    text: (x.text || '').toString(),
    imageFileId: parseInt(x.imageFileId || 0, 10) || 0,
    buttonText: (x.buttonText || '').toString(),
    buttonUrl: (x.buttonUrl || '').toString(),
  };
}

function cardsBuilderTemplate(columns) {
  if (columns === 2) return '1fr 1fr';
  if (columns === 4) return '1fr 1fr 1fr 1fr';
  return '1fr 1fr 1fr';
}

function cardsRenderBuilderItems(items, files, fileDownloadUrl) {
  const fileOptions = (selectedId) => {
    const opts = ['<option value="0">— без картинки —</option>'];
    files.forEach(f => {
      const s = (parseInt(f.id,10) === selectedId) ? 'selected' : '';
      opts.push(`<option value="${f.id}" ${s}>${BX.util.htmlspecialchars(f.name)} (id ${f.id})</option>`);
    });
    return opts.join('');
  };

  return items.map((it, idx) => {
    const title = BX.util.htmlspecialchars(it.title || '');
    const text = BX.util.htmlspecialchars(it.text || '');
    const btnText = BX.util.htmlspecialchars(it.buttonText || '');
    const btnUrl = BX.util.htmlspecialchars(it.buttonUrl || '');
    const imgId = parseInt(it.imageFileId || 0, 10) || 0;

    const imgPrev = imgId ? `<div class="imgPrev"><img src="${fileDownloadUrl(imgId)}" alt=""></div>` : '';

    return `
      <div class="item" data-ci="${idx}">
        <div class="itemHead">
          <div><b>Карточка ${idx + 1}</b></div>
          <div class="miniBtns">
            <button class="ui-btn ui-btn-light ui-btn-xs" data-card-up="${idx}">↑</button>
            <button class="ui-btn ui-btn-light ui-btn-xs" data-card-down="${idx}">↓</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-card-del="${idx}">Удалить</button>
          </div>
        </div>

        <div class="grid2" style="margin-top:10px;">
          <div>
            <div class="field">
              <label>Заголовок</label>
              <input class="input" data-card-title="${idx}" value="${title}">
            </div>
            <div class="field">
              <label>Текст</label>
              <textarea class="input" data-card-text="${idx}" style="height:120px;">${text}</textarea>
            </div>
          </div>

          <div>
            <div class="field">
              <label>Картинка</label>
              <select class="input" data-card-img="${idx}">
                ${fileOptions(imgId)}
              </select>
            </div>
            <div data-card-img-prev="${idx}">
              ${imgPrev}
            </div>

            <div class="field">
              <label>Текст кнопки (опц.)</label>
              <input class="input" data-card-btntext="${idx}" value="${btnText}">
            </div>
            <div class="field">
              <label>URL кнопки (опц.)</label>
              <input class="input" data-card-btnurl="${idx}" value="${btnUrl}">
            </div>
          </div>
        </div>
      </div>
    `;
  }).join('');
}

async function openCardsBuilderDialog(mode, blockId, currentContent, api, getFilesForSite, fileDownloadUrl, loadBlocks, notify, pageId) {
  let cols = currentContent?.columns ? parseInt(currentContent.columns, 10) : 3;
  if (![2,3,4].includes(cols)) cols = 3;

  let items = Array.isArray(currentContent?.items) ? currentContent.items.map(cardsNormalizeItem) : [];
  if (!items.length) items = [cardsNormalizeItem({title:'Преимущество 1'}), cardsNormalizeItem({title:'Преимущество 2'})];

  let files = [];
  try { files = await getFilesForSite(); } catch(e) { files = []; }

  const render = () => `
    <div class="cardsBuilder">
      <div class="field">
        <label>Колонки</label>
        <select id="cb_cols" class="input">
          <option value="2" ${cols===2?'selected':''}>2</option>
          <option value="3" ${cols===3?'selected':''}>3</option>
          <option value="4" ${cols===4?'selected':''}>4</option>
        </select>
      </div>

      <div style="margin-top:10px;">
        <button class="ui-btn ui-btn-light" id="cb_add">+ Добавить карточку</button>
      </div>

      <div id="cb_items">
        ${cardsRenderBuilderItems(items, files, fileDownloadUrl)}
      </div>

      <div class="muted" style="margin-top:10px;">
        Минимум у карточки должен быть заголовок. Картинка и кнопка — опционально.
      </div>
    </div>
  `;

  BX.UI.Dialogs.MessageBox.show({
    title: mode === 'edit' ? ('Редактировать Cards #' + blockId) : 'Новый Cards блок',
    message: render(),
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function(mb){
      cols = parseInt(document.getElementById('cb_cols')?.value || '3', 10);
      if (![2,3,4].includes(cols)) { notify('columns должен быть 2/3/4'); return; }

      const collected = items.map((_, idx) => {
        const title = (document.querySelector(`[data-card-title="${idx}"]`)?.value || '').trim();
        const text = (document.querySelector(`[data-card-text="${idx}"]`)?.value || '');
        const imageFileId = parseInt(document.querySelector(`[data-card-img="${idx}"]`)?.value || '0', 10) || 0;
        const buttonText = (document.querySelector(`[data-card-btntext="${idx}"]`)?.value || '').trim();
        const buttonUrl = (document.querySelector(`[data-card-btnurl="${idx}"]`)?.value || '').trim();
        return { title, text, imageFileId, buttonText, buttonUrl };
      }).filter(x => x.title !== '');

      if (!collected.length) { notify('Добавь хотя бы одну карточку с заголовком'); return; }

      const payload = { columns: cols, items: JSON.stringify(collected) };

      const call = (mode === 'edit')
        ? api('block.update', Object.assign({ id: blockId }, payload))
        : api('block.create', Object.assign({ pageId, type:'cards' }, payload));

      call.then(res => {
        if (!res || res.ok !== true) { notify('Не удалось сохранить cards'); return; }
        notify(mode === 'edit' ? 'Сохранено' : 'Cards создан');
        mb.close();
        loadBlocks();
      }).catch(() => notify('Ошибка запроса cards'));
    }
  });

  setTimeout(() => {
    const root = document.querySelector('.cardsBuilder');
    if (!root) return;

    const snapshot = () => {
      items = items.map((it, idx) => ({
        title: (document.querySelector(`[data-card-title="${idx}"]`)?.value || it.title || ''),
        text: (document.querySelector(`[data-card-text="${idx}"]`)?.value || it.text || ''),
        imageFileId: parseInt(document.querySelector(`[data-card-img="${idx}"]`)?.value || it.imageFileId || 0, 10) || 0,
        buttonText: (document.querySelector(`[data-card-btntext="${idx}"]`)?.value || it.buttonText || ''),
        buttonUrl: (document.querySelector(`[data-card-btnurl="${idx}"]`)?.value || it.buttonUrl || ''),
      }));
      cols = parseInt(document.getElementById('cb_cols')?.value || String(cols), 10);
      if (![2,3,4].includes(cols)) cols = 3;
    };

    const rerender = () => {
      snapshot();
      root.innerHTML = render();
      bind();
    };

    const bind = () => {
      const addBtn = document.getElementById('cb_add');
      if (addBtn) addBtn.onclick = () => { snapshot(); items.push(cardsNormalizeItem({ title: 'Новая карточка' })); rerender(); };

      root.querySelectorAll('[data-card-up]').forEach(btn => {
        btn.onclick = () => {
          snapshot();
          const i = parseInt(btn.getAttribute('data-card-up'), 10);
          if (i > 0) { const t = items[i-1]; items[i-1] = items[i]; items[i] = t; rerender(); }
        };
      });
      root.querySelectorAll('[data-card-down]').forEach(btn => {
        btn.onclick = () => {
          snapshot();
          const i = parseInt(btn.getAttribute('data-card-down'), 10);
          if (i < items.length - 1) { const t = items[i+1]; items[i+1] = items[i]; items[i] = t; rerender(); }
        };
      });
      root.querySelectorAll('[data-card-del]').forEach(btn => {
        btn.onclick = () => {
          snapshot();
          const i = parseInt(btn.getAttribute('data-card-del'), 10);
          items.splice(i, 1);
          if (!items.length) items.push(cardsNormalizeItem({ title: 'Новая карточка' }));
          rerender();
        };
      });

      root.querySelectorAll('select[data-card-img]').forEach(sel => {
        sel.onchange = () => {
          const idx = parseInt(sel.getAttribute('data-card-img'), 10);
          const fid = parseInt(sel.value || '0', 10);
          const box = root.querySelector(`[data-card-img-prev="${idx}"]`);
          if (!box) return;
          box.innerHTML = fid ? `<div class="imgPrev"><img src="${fileDownloadUrl(fid)}" alt=""></div>` : '';
        };
      });
    };

    bind();
  }, 0);
}
```

---

# ✅ Шаг 2. Исправь `addCardsBlock()` чтобы передавать зависимости

Твой `addCardsBlock()` сейчас вызывает `openCardsBuilderDialog(...)` без параметров — а мы сделали его безопасным и явным.

Замени `addCardsBlock()` на:

```js
function addCardsBlock() {
  openCardsBuilderDialog(
    'create',
    0,
    { columns: 3, items: [
      { title: 'Преимущество 1', text: 'Короткое описание' },
      { title: 'Преимущество 2', text: 'Короткое описание' },
      { title: 'Преимущество 3', text: 'Короткое описание' }
    ]},
    api, getFilesForSite, fileDownloadUrl, loadBlocks, notify, pageId
  );
}
```

А `editCardsBlock(id)` — на:

```js
function editCardsBlock(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
    openCardsBuilderDialog(
      'edit',
      id,
      blk?.content || null,
      api, getFilesForSite, fileDownloadUrl, loadBlocks, notify, pageId
    );
  }).catch(() => notify('Ошибка block.list'));
}
```

---

# ✅ Почему у тебя так вышло

Ты добавил `addCardsBlock()` и повесил кнопку, но **саму функцию `openCardsBuilderDialog` либо не вставил вообще**, либо вставил **после** места, где файл “ломается” синтаксически, и браузер до неё не дошёл.

---

## Быстрая проверка после правок

1. Обнови страницу редактора (Ctrl+F5)
2. В консоли не должно быть `openCardsBuilderDialog is not defined`
3. Нажми **+ Cards** — откроется диалог-конструктор

Если после вставки снова “ничего” — скажи, какая первая красная ошибка в консоли (обычно это syntax error выше по коду).
