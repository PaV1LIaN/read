Нашёл, где логика слабая: у тебя сохранение порядка завязано на drop, а это событие в таком DnD срабатывает нестабильно. Визуально блок двигается, но saveBlockOrder() не всегда вызывается в нужный момент. Поэтому после loadBlocks() всё откатывается.

Как починить надёжно

Нужно сохранять порядок в dragend, а не в drop.

Тогда схема будет такой:

dragstart — запомнили исходный порядок

dragover — двигаем DOM

dragend — сравнили порядок до/после, если изменился → вызвали saveBlockOrder()


Это гораздо надёжнее.


---

Что заменить

Полностью замени функцию initBlockDnD()

Найди в editor.php:

function initBlockDnD() {

И замени всю функцию целиком на это:

function initBlockDnD() {
  if (!blocksBox) return;

  let draggedEl = null;
  let dragAllowed = false;
  let startOrder = '';

  function currentOrderString() {
    return Array.from(blocksBox.querySelectorAll('[data-block-id]'))
      .map(el => String(el.getAttribute('data-block-id') || ''))
      .filter(Boolean)
      .join(',');
  }

  blocksBox.querySelectorAll('[data-block-id]').forEach(block => {
    block.setAttribute('draggable', 'false');

    const handle = block.querySelector('[data-drag-handle]');
    if (handle) {
      handle.addEventListener('mousedown', () => {
        dragAllowed = true;
        block.setAttribute('draggable', 'true');
      });

      handle.addEventListener('mouseup', () => {
        setTimeout(() => {
          block.setAttribute('draggable', 'false');
          dragAllowed = false;
        }, 0);
      });
    }

    block.addEventListener('dragstart', (e) => {
      if (!dragAllowed) {
        e.preventDefault();
        return;
      }

      draggedEl = block;
      startOrder = currentOrderString();
      block.classList.add('dragging');

      try {
        e.dataTransfer.effectAllowed = 'move';
        e.dataTransfer.setData('text/plain', block.getAttribute('data-block-id') || '');
      } catch (err) {}
    });

    block.addEventListener('dragover', (e) => {
      if (!draggedEl || draggedEl === block) return;
      e.preventDefault();

      const rect = block.getBoundingClientRect();
      const middle = rect.top + rect.height / 2;
      const after = e.clientY > middle;

      if (after) {
        if (block.nextElementSibling !== draggedEl) {
          block.parentNode.insertBefore(draggedEl, block.nextElementSibling);
        }
      } else {
        if (block.previousElementSibling !== draggedEl) {
          block.parentNode.insertBefore(draggedEl, block);
        }
      }
    });

    block.addEventListener('drop', (e) => {
      e.preventDefault();
    });

    block.addEventListener('dragend', async () => {
      const changed = startOrder && startOrder !== currentOrderString();

      if (draggedEl) draggedEl.classList.remove('dragging');

      draggedEl = null;
      dragAllowed = false;
      block.setAttribute('draggable', 'false');

      if (!changed) return;

      try {
        const r = await saveBlockOrder();
        if (!r || r.ok !== true) {
          notify('Не удалось сохранить порядок');
          loadBlocks();
          return;
        }
        notify('Порядок сохранён');
        loadBlocks();
      } catch (err) {
        notify('Ошибка block.reorder');
        loadBlocks();
      }
    });
  });
}


---

Ещё одна важная правка

Найди saveBlockOrder()

Сейчас там, скорее всего:

const ids = Array.from(blocksBox.querySelectorAll('.block[data-block-id]'))

Замени на:

const ids = Array.from(blocksBox.querySelectorAll('[data-block-id]'))

Полностью функция должна выглядеть так:

function saveBlockOrder() {
  const ids = Array.from(blocksBox.querySelectorAll('[data-block-id]'))
    .map(el => parseInt(el.getAttribute('data-block-id'), 10))
    .filter(Boolean);

  return api('block.reorder', {
    pageId,
    order: JSON.stringify(ids)
  });
}


---

Почему это исправит проблему

Было

Сохранение шло только через drop, а оно ненадёжно.

Стало

Сохранение идёт в dragend, когда DOM уже точно перестроен.


---

Что проверить после правки

1.

Перетащи обычный блок — после отпускания должен появиться Порядок сохранён

2.

Перетащи section — тоже должен сохраниться

3.

Обнови страницу редактора — порядок не должен откатываться

4.

Если всё ещё не сохранится Тогда уже проблема не во фронте, а в ответе api.php, и нужно будет смотреть конкретный ответ block.reorder из Network.

Но по текущему коду самая вероятная поломка именно здесь.