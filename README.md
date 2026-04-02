Идём дальше.

Создай файл:

/local/sitebuilder/assets/js/editor.dnd.js

и вставь туда это:

window.SBEditor = window.SBEditor || {};

window.SBEditor.saveBlockOrder = function (orderedIdsFromDom = null) {
  const st = window.SBEditor.getState();

  let ids = Array.isArray(orderedIdsFromDom)
    ? orderedIdsFromDom.slice()
    : Array.from(st.blocksBox.querySelectorAll('[data-block-id]'))
        .map(el => parseInt(el.getAttribute('data-block-id'), 10))
        .filter(Boolean);

  const knownIds = new Set(ids);

  for (const b of (st.allPageBlocks || [])) {
    const bid = parseInt(b.id, 10);
    if (bid > 0 && !knownIds.has(bid)) {
      ids.push(bid);
      knownIds.add(bid);
    }
  }

  return window.SBEditor.api('block.reorder', {
    pageId: st.pageId,
    order: JSON.stringify(ids)
  });
};

window.SBEditor.initBlockDnD = function () {
  const st = window.SBEditor.getState();
  const blocksBox = st.blocksBox;
  const blockSearch = st.blockSearch;

  if (!blocksBox) return;

  const searchValue = (blockSearch?.value || '').trim();
  if (searchValue !== '') return;

  let draggedEl = null;
  let dragAllowed = false;
  let startOrder = '';

  function currentOrderedIds() {
    return Array.from(blocksBox.querySelectorAll('[data-block-id]'))
      .map(el => parseInt(el.getAttribute('data-block-id') || '0', 10))
      .filter(Boolean);
  }

  function currentOrderString() {
    return currentOrderedIds().join(',');
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
        const r = await window.SBEditor.saveBlockOrder(currentOrderedIds());
        if (!r || r.ok !== true) {
          window.SBEditor.notify('Не удалось сохранить порядок блоков');
          console.error('block.reorder failed', r);
          if (typeof window.SBEditor.loadBlocks === 'function') {
            window.SBEditor.loadBlocks();
          }
          return;
        }

        window.SBEditor.notify('Порядок сохранён');

        if (typeof window.SBEditor.loadBlocks === 'function') {
          window.SBEditor.loadBlocks();
        }
      } catch (err) {
        window.SBEditor.notify('Ошибка block.reorder');
        console.error(err);

        if (typeof window.SBEditor.loadBlocks === 'function') {
          window.SBEditor.loadBlocks();
        }
      }
    });
  });
};

Теперь в editor.core.js добавь ещё поле состояния, если не добавил:

allPageBlocks: [],

То есть state должен содержать:

allPageBlocks: [],

Следующий файл — editor.blocks.js. В нём будет:

buildBlockShell

renderBlocks

логика section

обвязка блоков внутри section


Напиши дальше, и я дам его.