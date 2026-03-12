Сейчас причина почти наверняка в том, что draggable="true" повешен на весь .block, а браузер не даёт нормально стартовать drag только через наш closest('[data-drag-handle]').

Сделаем надёжнее:
перетаскивание включается только когда зажали ⋮⋮.

Что заменить в editor.php

Найди функцию:

function initBlockDnD() {

и полностью замени её на эту:

function initBlockDnD() {
  if (!blocksBox) return;

  let draggedEl = null;
  let dragAllowed = false;

  blocksBox.querySelectorAll('.block[data-block-id]').forEach(block => {
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
      block.classList.add('dragging');

      try {
        e.dataTransfer.effectAllowed = 'move';
        e.dataTransfer.setData('text/plain', block.getAttribute('data-block-id') || '');
      } catch (err) {}
    });

    block.addEventListener('dragend', () => {
      if (draggedEl) draggedEl.classList.remove('dragging');
      draggedEl = null;
      dragAllowed = false;
      block.setAttribute('draggable', 'false');
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

    block.addEventListener('drop', async (e) => {
      if (!draggedEl) return;
      e.preventDefault();

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

Что ещё проверить

Убедись, что в buildBlockShell() есть handle:

<span class="dragHandle" data-drag-handle="${id}" title="Перетащить">⋮⋮</span>


---

Как тестировать

Тянуть нужно именно за ⋮⋮, не за весь блок.

Ожидаемое поведение:

блок начинает “прилипать” к курсору,

можно опустить выше/ниже,

потом нотификация Порядок сохранён.



---

Если всё равно не двигается

Тогда проблема уже не в UI, а в том, что браузер вообще не стартует drag.
В таком случае я дам тебе другой, ещё более надёжный вариант — сортировку на Pointer Events без HTML5 drag-and-drop, он обычно работает стабильнее.

Если после этой замены не заработает, пришли кусок из консоли браузера, если там есть ошибки.