Идём дальше — теперь делаем **визуальный конструктор Cards** (без JSON), чтобы можно было удобно:

* добавлять/удалять карточки,
* менять порядок ↑↓,
* выбирать картинку из “Файлы”,
* править заголовок/текст/кнопку.

Ниже даю **патч только для `editor.php`** (API и view у тебя уже готовы под `cards`).

---

# 1) editor.php — что добавляем

## 1.1 CSS (для конструктора)

В `<style>` добавь:

```css
/* cards builder */
.cardsBuilder { margin-top:10px; }
.cardsBuilder .item { border:1px solid #e5e7ea; border-radius:12px; padding:10px; margin-top:10px; background:#fff; }
.cardsBuilder .itemHead { display:flex; gap:8px; align-items:center; justify-content:space-between; flex-wrap:wrap; }
.cardsBuilder .miniBtns { display:flex; gap:6px; flex-wrap:wrap; }
.cardsBuilder .grid2 { display:grid; grid-template-columns: 1fr 1fr; gap:10px; }
@media (max-width: 720px){ .cardsBuilder .grid2 { grid-template-columns: 1fr; } }
```

---

## 1.2 renderBlocks: чтобы “Редактировать” у cards вызывало визуальный конструктор

Если у тебя сейчас `cards` ещё в JSON-варианте — оставь превью как есть, но важно чтобы кнопка была:

```html
<button ... data-edit-cards-id="${id}">Редактировать</button>
```

(скорее всего уже есть)

---

# 2) editor.php — вставь функции конструктора

Добавь эти функции в JS (лучше рядом с другими `open*Dialog`).

### 2.1 Хелперы для Cards Builder

```js
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

function cardsRenderBuilderItems(items, files) {
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
```

### 2.2 Основной диалог-конструктор

```js
async function openCardsBuilderDialog(mode, blockId, currentContent) {
  const currentCols = currentContent?.columns ? parseInt(currentContent.columns, 10) : 3;
  let items = Array.isArray(currentContent?.items) ? currentContent.items.map(cardsNormalizeItem) : [];
  if (!items.length) items = [cardsNormalizeItem({title:'Преимущество 1'}), cardsNormalizeItem({title:'Преимущество 2'})];

  // заранее загрузим файлы (для селектов)
  let files = [];
  try { files = await getFilesForSite(); } catch(e) { files = []; }

  // локальный "рендер" диалога: будем обновлять innerHTML при изменениях
  const render = () => `
    <div class="cardsBuilder">
      <div class="field">
        <label>Колонки</label>
        <select id="cb_cols" class="input">
          <option value="2" ${currentCols===2?'selected':''}>2</option>
          <option value="3" ${currentCols===3?'selected':''}>3</option>
          <option value="4" ${currentCols===4?'selected':''}>4</option>
        </select>
      </div>

      <div style="margin-top:10px;">
        <button class="ui-btn ui-btn-light" id="cb_add">+ Добавить карточку</button>
      </div>

      <div id="cb_items">
        ${cardsRenderBuilderItems(items, files)}
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
      // собираем данные из инпутов
      const cols = parseInt(document.getElementById('cb_cols')?.value || '3', 10);
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

  // навешиваем события после вставки DOM
  setTimeout(() => {
    const root = document.querySelector('.cardsBuilder');
    if (!root) return;

    const rerender = () => {
      // сохраняем текущие значения перед перерисовкой
      items = items.map((it, idx) => ({
        title: (document.querySelector(`[data-card-title="${idx}"]`)?.value || it.title || ''),
        text: (document.querySelector(`[data-card-text="${idx}"]`)?.value || it.text || ''),
        imageFileId: parseInt(document.querySelector(`[data-card-img="${idx}"]`)?.value || it.imageFileId || 0, 10) || 0,
        buttonText: (document.querySelector(`[data-card-btntext="${idx}"]`)?.value || it.buttonText || ''),
        buttonUrl: (document.querySelector(`[data-card-btnurl="${idx}"]`)?.value || it.buttonUrl || ''),
      }));

      root.innerHTML = render();

      // снова подключаем слушатели
      bind();
    };

    const bind = () => {
      const addBtn = document.getElementById('cb_add');
      if (addBtn) {
        addBtn.onclick = () => {
          items.push(cardsNormalizeItem({ title: 'Новая карточка', text: '' }));
          rerender();
        };
      }

      // up/down/delete
      root.querySelectorAll('[data-card-up]').forEach(btn => {
        btn.onclick = () => {
          const i = parseInt(btn.getAttribute('data-card-up'), 10);
          if (i > 0) { const t = items[i-1]; items[i-1] = items[i]; items[i] = t; rerender(); }
        };
      });
      root.querySelectorAll('[data-card-down]').forEach(btn => {
        btn.onclick = () => {
          const i = parseInt(btn.getAttribute('data-card-down'), 10);
          if (i < items.length - 1) { const t = items[i+1]; items[i+1] = items[i]; items[i] = t; rerender(); }
        };
      });
      root.querySelectorAll('[data-card-del]').forEach(btn => {
        btn.onclick = () => {
          const i = parseInt(btn.getAttribute('data-card-del'), 10);
          items.splice(i, 1);
          if (!items.length) items.push(cardsNormalizeItem({ title: 'Новая карточка' }));
          rerender();
        };
      });

      // превью картинки при выборе
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

### 2.3 add/edit функции (cards)

Заменяем JSON-вариант (или просто переопределяем) на визуальный:

```js
function addCardsBlock() {
  openCardsBuilderDialog('create', 0, { columns: 3, items: [
    { title: 'Преимущество 1', text: 'Короткое описание' },
    { title: 'Преимущество 2', text: 'Короткое описание' },
    { title: 'Преимущество 3', text: 'Короткое описание' }
  ]});
}

function editCardsBlock(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
    openCardsBuilderDialog('edit', id, blk?.content || null);
  }).catch(() => notify('Ошибка block.list'));
}
```

---

# 3) Подключение событий

## 3.1 Кнопка “+ Cards”

Убедись, что у тебя есть:

```js
btnAddCards.addEventListener('click', addCardsBlock);
```

## 3.2 Click-handler для редактирования

В общем обработчике кликов добавь:

```js
const ecards = e.target.closest('[data-edit-cards-id]');
if (ecards) { editCardsBlock(parseInt(ecards.getAttribute('data-edit-cards-id'), 10)); return; }
```

---

# Что проверить

1. Нажать **+ Cards** → откроется конструктор, добавится 3 карточки по умолчанию
2. Поменять заголовки, добавить картинку (если есть файлы), поставить кнопку
3. Поменять порядок ↑↓, удалить карточку
4. OK → в списке блоков “cards” обновится `items count`, а на `view.php` будет сетка

---

Если это заработает, следующий блок я бы сделал **Form block** (форма заявки с отправкой письма / создание лида в CRM Bitrix24). Это уже “бизнес-функция”, а не просто верстка.
