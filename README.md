Часть 5 из 8:

const items = (b.content && Array.isArray(b.content.items)) ? b.content.items : [];
        const tpl = galleryTemplate(columns);

        const cardsHtml = items.map(raw => {
          const it = cardsNormalizeItem(raw || {});
          const img = it.imageFileId ? `<div class="imgPrev"><img src="${fileDownloadUrl(it.imageFileId)}" alt=""></div>` : '';

          return `
            <div class="miniCard">
              <div style="font-weight:700;">${BX.util.htmlspecialchars(it.title)}</div>
              <div class="muted" style="margin-top:6px; white-space:pre-wrap;">${BX.util.htmlspecialchars(it.text)}</div>
              ${img}
              ${it.buttonUrl ? `<div class="muted" style="margin-top:6px;">${BX.util.htmlspecialchars(it.buttonText || 'Открыть')} → ${BX.util.htmlspecialchars(it.buttonUrl)}</div>` : ''}
            </div>
          `;
        }).join('');

        const wrap = wrapBlockMeta();
        return buildBlockShell(
          id, type, sort,
          `<div class="cardsPrev" style="grid-template-columns:${tpl};">${cardsHtml}</div>`,
          `
            ${commonBtns}
            <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cards-id="${id}">Редактировать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
          `,
          `<span>cols: ${columns}</span><span>items: ${items.length}</span>`,
          '',
          wrap.extraClass,
          wrap.sectionMarkHtml
        );
      }

      const wrap = wrapBlockMeta();
      return buildBlockShell(
        id, type, sort,
        `<div class="muted">Тип блока не поддержан: ${BX.util.htmlspecialchars(type)}</div>`,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        '',
        '',
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }).join('');

    initBlockDnD();
  }

  function saveBlockOrder(orderedIdsFromDom = null) {
    let ids = Array.isArray(orderedIdsFromDom)
      ? orderedIdsFromDom.slice()
      : Array.from(blocksBox.querySelectorAll('[data-block-id]'))
          .map(el => parseInt(el.getAttribute('data-block-id'), 10))
          .filter(Boolean);

    const knownIds = new Set(ids);

    for (const b of allPageBlocks) {
      const bid = parseInt(b.id, 10);
      if (bid > 0 && !knownIds.has(bid)) {
        ids.push(bid);
        knownIds.add(bid);
      }
    }

    return api('block.reorder', {
      pageId,
      order: JSON.stringify(ids)
    });
  }

  function initBlockDnD() {
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
          const r = await saveBlockOrder(currentOrderedIds());
          if (!r || r.ok !== true) {
            notify('Не удалось сохранить порядок блоков');
            console.error('block.reorder failed', r);
            loadBlocks();
            return;
          }

          notify('Порядок сохранён');
          loadBlocks();
        } catch (err) {
          notify('Ошибка block.reorder');
          console.error(err);
          loadBlocks();
        }
      });
    });
  }

  const SECTION_PRESETS = {
    default: {
      boxed: true,
      background: '#FFFFFF',
      paddingTop: 32,
      paddingBottom: 32,
      border: false,
      radius: 0
    },
    hero: {
      boxed: true,
      background: '#F8FAFC',
      paddingTop: 72,
      paddingBottom: 72,
      border: false,
      radius: 0
    },
    light: {
      boxed: true,
      background: '#F9FAFB',
      paddingTop: 40,
      paddingBottom: 40,
      border: false,
      radius: 0
    },
    accent: {
      boxed: false,
      background: '#EEF2FF',
      paddingTop: 56,
      paddingBottom: 56,
      border: false,
      radius: 0
    },
    card: {
      boxed: true,
      background: '#FFFFFF',
      paddingTop: 32,
      paddingBottom: 32,
      border: true,
      radius: 16
    }
  };

Пришлю часть 6 следующим сообщением.