Делаем. Но прямо сейчас точечно по файлу не смогу показать места вставки: загруженный архив уже протух, и открыть текущий editor.php снова нельзя.

Чтобы не тормозить, даю готовую схему, что добавить. Если потом перезальёшь архив, я укажу exact-места по строкам.

Что делаем

У карточки section добавим быстрые кнопки:

+ Heading

+ Text

+ Button

+ Cards


Они будут создавать обычный блок сразу после этой section.


---

1. Добавь helper для вставки блока после section

В <script> editor.php, рядом с addSectionBlock() и другими add...Block() функциями, вставь:

async function createBlockAfterSection(sectionId, type, payload = {}) {
  const listRes = await api('block.list', { pageId });
  if (!listRes || listRes.ok !== true) {
    notify('Не удалось загрузить блоки страницы');
    return;
  }

  const blocks = Array.isArray(listRes.blocks) ? listRes.blocks.slice() : [];
  const sectionIndex = blocks.findIndex(b => parseInt(b.id, 10) === parseInt(sectionId, 10));

  if (sectionIndex < 0) {
    notify('Section не найдена');
    return;
  }

  const sectionSort = parseInt(blocks[sectionIndex].sort || 0, 10);

  let insertSort = sectionSort + 10;
  for (let i = sectionIndex + 1; i < blocks.length; i++) {
    const next = blocks[i];
    if ((next.type || '') === 'section') {
      insertSort = parseInt(next.sort || insertSort, 10) - 1;
      break;
    }
    insertSort = Math.max(insertSort, parseInt(next.sort || 0, 10) + 10);
  }

  const createRes = await api('block.create', {
    pageId,
    type,
    ...payload
  });

  if (!createRes || createRes.ok !== true || !createRes.block) {
    notify('Не удалось создать блок');
    return;
  }

  const newBlockId = parseInt(createRes.block.id, 10);

  const refreshRes = await api('block.list', { pageId });
  if (!refreshRes || refreshRes.ok !== true) {
    notify('Блок создан, но не удалось обновить порядок');
    loadBlocks();
    return;
  }

  const freshBlocks = Array.isArray(refreshRes.blocks) ? refreshRes.blocks.slice() : [];
  const moved = freshBlocks.find(b => parseInt(b.id, 10) === newBlockId);
  if (!moved) {
    loadBlocks();
    return;
  }

  const desiredOrder = freshBlocks
    .sort((a, b) => parseInt(a.sort || 0, 10) - parseInt(b.sort || 0, 10))
    .filter(b => parseInt(b.id, 10) !== newBlockId);

  let targetIndex = desiredOrder.findIndex(b => parseInt(b.sort || 0, 10) > insertSort);
  if (targetIndex < 0) targetIndex = desiredOrder.length;

  desiredOrder.splice(targetIndex, 0, moved);

  const order = desiredOrder.map(b => parseInt(b.id, 10));

  const reorderRes = await api('block.reorder', {
    pageId,
    order: JSON.stringify(order)
  });

  if (!reorderRes || reorderRes.ok !== true) {
    notify('Блок создан, но не удалось поставить после section');
    loadBlocks();
    return;
  }

  notify('Блок добавлен');
  loadBlocks();
}


---

2. Добавь быстрые функции для типовых блоков

Сразу после helper выше вставь:

function quickAddHeadingAfterSection(sectionId) {
  createBlockAfterSection(sectionId, 'heading', {
    text: 'Новый заголовок',
    level: 'h2',
    align: 'left'
  });
}

function quickAddTextAfterSection(sectionId) {
  createBlockAfterSection(sectionId, 'text', {
    text: 'Новый текст'
  });
}

function quickAddButtonAfterSection(sectionId) {
  createBlockAfterSection(sectionId, 'button', {
    text: 'Кнопка',
    url: '/',
    variant: 'primary'
  });
}

function quickAddCardsAfterSection(sectionId) {
  createBlockAfterSection(sectionId, 'cards', {
    columns: 3,
    items: JSON.stringify([
      { title: 'Карточка 1', text: 'Описание 1', imageFileId: 0, buttonText: '', buttonUrl: '' },
      { title: 'Карточка 2', text: 'Описание 2', imageFileId: 0, buttonText: '', buttonUrl: '' },
      { title: 'Карточка 3', text: 'Описание 3', imageFileId: 0, buttonText: '', buttonUrl: '' }
    ])
  });
}


---

3. Добавь кнопки в рендер section

В ветке:

if (type === 'section') {

где у тебя сейчас кнопки section (Редактировать, Дублировать, Удалить, стрелки), добавь рядом ещё этот кусок:

<button class="ui-btn ui-btn-success ui-btn-xs" data-add-heading-after-section-id="${id}">+ Heading</button>
<button class="ui-btn ui-btn-success ui-btn-xs" data-add-text-after-section-id="${id}">+ Text</button>
<button class="ui-btn ui-btn-success ui-btn-xs" data-add-button-after-section-id="${id}">+ Button</button>
<button class="ui-btn ui-btn-success ui-btn-xs" data-add-cards-after-section-id="${id}">+ Cards</button>

Если хочешь аккуратнее, можно обернуть их в отдельный блок:

<div class="btns" style="margin-top:8px;">
  <button class="ui-btn ui-btn-success ui-btn-xs" data-add-heading-after-section-id="${id}">+ Heading</button>
  <button class="ui-btn ui-btn-success ui-btn-xs" data-add-text-after-section-id="${id}">+ Text</button>
  <button class="ui-btn ui-btn-success ui-btn-xs" data-add-button-after-section-id="${id}">+ Button</button>
  <button class="ui-btn ui-btn-success ui-btn-xs" data-add-cards-after-section-id="${id}">+ Cards</button>
</div>


---

4. Подключи обработчики клика

В общем document.addEventListener('click', ...) добавь:

const addHeadingAfterSectionBtn = e.target.closest('[data-add-heading-after-section-id]');
if (addHeadingAfterSectionBtn) {
  quickAddHeadingAfterSection(parseInt(addHeadingAfterSectionBtn.getAttribute('data-add-heading-after-section-id'), 10));
  return;
}

const addTextAfterSectionBtn = e.target.closest('[data-add-text-after-section-id]');
if (addTextAfterSectionBtn) {
  quickAddTextAfterSection(parseInt(addTextAfterSectionBtn.getAttribute('data-add-text-after-section-id'), 10));
  return;
}

const addButtonAfterSectionBtn = e.target.closest('[data-add-button-after-section-id]');
if (addButtonAfterSectionBtn) {
  quickAddButtonAfterSection(parseInt(addButtonAfterSectionBtn.getAttribute('data-add-button-after-section-id'), 10));
  return;
}

const addCardsAfterSectionBtn = e.target.closest('[data-add-cards-after-section-id]');
if (addCardsAfterSectionBtn) {
  quickAddCardsAfterSection(parseInt(addCardsAfterSectionBtn.getAttribute('data-add-cards-after-section-id'), 10));
  return;
}


---

5. Что проверить

После этого проверь:

1. У SECTION появились зелёные быстрые кнопки.


2. Нажимаешь + Heading — заголовок создаётся сразу после section.


3. Нажимаешь + Text — текст тоже сразу под section.


4. Если на странице несколько секций, блок попадает под нужную секцию, а не в конец страницы.




---

Если хочешь, перезалей архив ещё раз, и я покажу exact-места вставки в твоём текущем editor.php без ручного поиска.