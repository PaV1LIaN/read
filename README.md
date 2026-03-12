Давай.

Я бы улучшил editor.php в таком порядке, чтобы стало заметно удобнее без перелома всей логики:

Что имеет смысл сделать сейчас

1. Липкую верхнюю панель Чтобы кнопки добавления блоков всегда были под рукой.


2. Свернуть/развернуть блок Сейчас список быстро разрастается, и редактор становится длинным.


3. Более компактные карточки блоков Сделать шапку блока аккуратнее: тип, id, sort, статус, кнопки.


4. Быстрый поиск по блокам Например по типу: text, image, card, gallery.


5. Подсветка разных типов блоков Чтобы визуально легче отличать text, gallery, cards, card.



Самый полезный старт — это сворачивание блоков + липкая панель + поиск.
Это сразу делает editor.php удобнее даже без новых API.


---

Предлагаю сделать первый пакет улучшений

1. Улучшаем стили верхней панели

В style замени стили .top и добавь новые:

.top {
  position: sticky;
  top: 0;
  z-index: 50;
  background: rgba(255,255,255,.94);
  backdrop-filter: blur(10px);
  border-bottom:1px solid #e5e7ea;
  padding:12px 16px;
  display:flex;
  gap:10px;
  align-items:center;
  flex-wrap:wrap;
}

.topActions {
  display:flex;
  gap:8px;
  flex-wrap:wrap;
  align-items:center;
}

.toolbarSearch {
  min-width: 220px;
  flex: 1;
  max-width: 360px;
}

.block {
  border:1px solid #e5e7ea;
  border-radius:14px;
  padding:12px;
  margin-top:12px;
  background:#fff;
  box-shadow: 0 1px 2px rgba(0,0,0,.03);
}

.blockHeader {
  display:flex;
  gap:10px;
  align-items:flex-start;
  justify-content:space-between;
  flex-wrap:wrap;
}

.blockLeft {
  display:flex;
  flex-direction:column;
  gap:6px;
  min-width: 220px;
}

.blockTitleRow {
  display:flex;
  gap:8px;
  align-items:center;
  flex-wrap:wrap;
}

.blockTypeBadge {
  display:inline-flex;
  align-items:center;
  padding:3px 8px;
  border-radius:999px;
  font-size:11px;
  font-weight:700;
  border:1px solid #e5e7ea;
  background:#f8fafc;
  color:#334155;
}

.blockMeta {
  display:flex;
  gap:8px;
  flex-wrap:wrap;
  color:#6a737f;
  font-size:12px;
}

.blockBody {
  margin-top:10px;
}

.blockCollapsed .blockBody {
  display:none;
}

.block[data-type="text"] .blockTypeBadge { background:#eff6ff; color:#1d4ed8; border-color:#bfdbfe; }
.block[data-type="image"] .blockTypeBadge { background:#f5f3ff; color:#6d28d9; border-color:#ddd6fe; }
.block[data-type="button"] .blockTypeBadge { background:#ecfeff; color:#0f766e; border-color:#a5f3fc; }
.block[data-type="heading"] .blockTypeBadge { background:#fef3c7; color:#92400e; border-color:#fcd34d; }
.block[data-type="columns2"] .blockTypeBadge { background:#f3f4f6; color:#374151; border-color:#d1d5db; }
.block[data-type="gallery"] .blockTypeBadge { background:#fdf2f8; color:#be185d; border-color:#fbcfe8; }
.block[data-type="spacer"] .blockTypeBadge { background:#f8fafc; color:#475569; border-color:#cbd5e1; }
.block[data-type="card"] .blockTypeBadge { background:#eef2ff; color:#3730a3; border-color:#c7d2fe; }
.block[data-type="cards"] .blockTypeBadge { background:#ecfccb; color:#3f6212; border-color:#bef264; }


---

2. Добавь поиск по блокам в верхнюю часть

Внутри .top, рядом с кнопками, добавь поле:

<div class="toolbarSearch">
  <input class="input" id="blockSearch" placeholder="Поиск по типу блока: text, image, card..." />
</div>


---

3. Перепиши рендер карточек блоков в более аккуратный формат

Вместо старого внешнего контейнера каждого блока используй такой шаблон.

В начале renderBlocks(blocks) добавь:

const q = (document.getElementById('blockSearch')?.value || '').trim().toLowerCase();
const filteredBlocks = (blocks || []).filter(b => {
  if (!q) return true;
  const type = String(b.type || '').toLowerCase();
  return type.includes(q) || String(b.id || '').includes(q);
});

if (!filteredBlocks.length) {
  blocksBox.innerHTML = '<div class="muted">Ничего не найдено.</div>';
  return;
}

И дальше рендерить уже filteredBlocks.map(...), а не blocks.map(...).


---

4. Сделай общую шапку блока

В каждом return для блока сейчас у тебя начинается с:

<div class="block">
  <div class="row">

Замени эту логику на такую структуру:

<div class="block" data-type="TYPE_HERE" data-block-id="ID_HERE">
  <div class="blockHeader">
    <div class="blockLeft">
      <div class="blockTitleRow">
        <b>#ID_HERE</b>
        <span class="blockTypeBadge">TYPE_HERE</span>
      </div>
      <div class="blockMeta">
        <span>sort: SORT_HERE</span>
      </div>
    </div>

    <div class="btns">
      <button class="ui-btn ui-btn-light ui-btn-xs" data-toggle-block="ID_HERE">Свернуть</button>
      ...
кнопки...
    </div>
  </div>

  <div class="blockBody">
    ... текущее содержимое блока ...
  </div>
</div>


---

5. Добавь сворачивание/разворачивание блоков

В BX.ready(function () { ... }) рядом с обработчиками добавь:

const collapsedBlocks = new Set();
const blockSearch = document.getElementById('blockSearch');

Ниже, в обработчик document.addEventListener('click', function (e) { ... }), добавь:

const tg = e.target.closest('[data-toggle-block]');
if (tg) {
  const id = parseInt(tg.getAttribute('data-toggle-block'), 10);
  const box = document.querySelector(`.block[data-block-id="${id}"]`);
  if (!box) return;

  if (collapsedBlocks.has(id)) {
    collapsedBlocks.delete(id);
    box.classList.remove('blockCollapsed');
    tg.textContent = 'Свернуть';
  } else {
    collapsedBlocks.add(id);
    box.classList.add('blockCollapsed');
    tg.textContent = 'Развернуть';
  }
  return;
}


---

6. После loadBlocks() привяжи поиск

Внизу, рядом с:

btnAddText.addEventListener('click', addTextBlock);

добавь:

if (blockSearch) {
  blockSearch.addEventListener('input', loadBlocks);
}


---

Что получится после этого

После этих правок editor.php станет:

заметно чище визуально,

удобнее на длинных страницах,

можно быстро найти нужный тип блока,

можно сворачивать тяжёлые блоки типа cards, gallery, card.



---

Как я бы сделал дальше

После этого следующим пакетом я бы улучшил именно карточки блоков:

вынес sort в маленький бейдж,

добавил дублировать блок,

добавил поднять в самый верх / вниз,

сделал gallery/cards компактнее.


Если хочешь, следующим сообщением я соберу тебе готовый обновлённый renderBlocks() целиком, чтобы ты просто заменил функцию.