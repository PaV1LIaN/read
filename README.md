Часть 2 из 8:

<div class="cell"><pre>${BX.util.htmlspecialchars(left)}</pre></div>
              <div class="cell"><pre>${BX.util.htmlspecialchars(right)}</pre></div>
            </div>
          `,
          `
            ${commonBtns}
            <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-columns2-id="${id}">Редактировать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
          `,
          `<span>ratio: ${BX.util.htmlspecialchars(ratio)}</span>`,
          '',
          wrap.extraClass,
          wrap.sectionMarkHtml
        );
      }

      if (type === 'gallery') {
        const columns = b.content && b.content.columns ? parseInt(b.content.columns, 10) : 3;
        const images = Array.isArray(b.content && b.content.images) ? b.content.images : [];
        const tpl = galleryTemplate(columns);

        const items = images.map(it => {
          const fid = parseInt(it.fileId || 0, 10);
          const alt = typeof it.alt === 'string' ? it.alt : '';
          return fid
            ? `<div class="imgPrev"><img src="${fileDownloadUrl(fid)}" alt="${BX.util.htmlspecialchars(alt)}"></div>`
            : '';
        }).join('');

        const wrap = wrapBlockMeta();
        return buildBlockShell(
          id, type, sort,
          `
            <div class="galleryPreview" style="grid-template-columns:${tpl};">
              ${items || '<div class="muted">Нет изображений</div>'}
            </div>
          `,
          `
            ${commonBtns}
            <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-gallery-id="${id}">Редактировать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
          `,
          `<span>columns: ${columns}</span><span>items: ${images.length}</span>`,
          '',
          wrap.extraClass,
          wrap.sectionMarkHtml
        );
      }

      if (type === 'spacer') {
        const height = b.content && b.content.height ? parseInt(b.content.height, 10) : 40;
        const line = !!(b.content && b.content.line);

        const wrap = wrapBlockMeta();
        return buildBlockShell(
          id, type, sort,
          `
            <div class="spacerPreview" style="height:${height}px;">
              ${line ? '<div class="spacerLine"></div>' : ''}
            </div>
          `,
          `
            ${commonBtns}
            <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-spacer-id="${id}">Редактировать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
          `,
          `<span>height: ${height}</span><span>line: ${line ? 'yes' : 'no'}</span>`,
          '',
          wrap.extraClass,
          wrap.sectionMarkHtml
        );
      }

      if (type === 'card') {
        const title = (b.content && typeof b.content.title === 'string') ? b.content.title : '';
        const text = (b.content && typeof b.content.text === 'string') ? b.content.text : '';
        const imageFileId = b.content && b.content.imageFileId ? parseInt(b.content.imageFileId, 10) : 0;
        const buttonText = (b.content && typeof b.content.buttonText === 'string') ? b.content.buttonText : '';
        const buttonUrl = (b.content && typeof b.content.buttonUrl === 'string') ? b.content.buttonUrl : '';

        const imageHtml = imageFileId
          ? `<div class="imgPrev"><img src="${fileDownloadUrl(imageFileId)}" alt=""></div>`
          : '';

        const wrap = wrapBlockMeta();
        return buildBlockShell(
          id, type, sort,
          `
            <div class="cardPreview">
              <div class="cardTitle">${BX.util.htmlspecialchars(title)}</div>
              ${text ? `<pre>${BX.util.htmlspecialchars(text)}</pre>` : ''}
              ${imageHtml}
              ${buttonUrl ? `<div class="muted">button: ${BX.util.htmlspecialchars(buttonText || 'Открыть')} → ${BX.util.htmlspecialchars(buttonUrl)}</div>` : ''}
            </div>
          `,
          `
            ${commonBtns}
            <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-card-id="${id}">Редактировать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
          `,
          `<span>image: ${imageFileId || '-'}</span>`,
          '',
          wrap.extraClass,
          wrap.sectionMarkHtml
        );
      }

      if (type === 'cards') {
        const columns = b.content && b.content.columns ? parseInt(b.content.columns, 10) : 3;
        const items = Array.isArray(b.content && b.content.items) ? b.content.items.map(cardsNormalizeItem) : [];
        const tpl = galleryTemplate(columns);

        const cardsHtml = items.map(it => {
          const imageHtml = it.imageFileId
            ? `<div class="imgPrev"><img src="${fileDownloadUrl(it.imageFileId)}" alt=""></div>`
            : '';

          return `
            <div class="cardPreview">
              <div class="cardTitle">${BX.util.htmlspecialchars(it.title)}</div>
              ${it.text ? `<pre>${BX.util.htmlspecialchars(it.text)}</pre>` : ''}
              ${imageHtml}
              ${it.buttonUrl ? `<div class="muted">button: ${BX.util.htmlspecialchars(it.buttonText || 'Открыть')} → ${BX.util.htmlspecialchars(it.buttonUrl)}</div>` : ''}
            </div>
          `;
        }).join('');

        const wrap = wrapBlockMeta();
        return buildBlockShell(
          id, type, sort,
          `
            <div class="galleryPreview" style="grid-template-columns:${tpl};">
              ${cardsHtml || '<div class="muted">Нет карточек</div>'}
            </div>
          `,
          `
            ${commonBtns}
            <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cards-id="${id}">Редактировать</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
          `,
          `<span>columns: ${columns}</span><span>items: ${items.length}</span>`,
          '',
          wrap.extraClass,
          wrap.sectionMarkHtml
        );
      }

      const wrap = wrapBlockMeta();
      return buildBlockShell(
        id, type, sort,
        `<div class="muted">Тип блока пока не поддержан в редакторе предпросмотра.</div>`,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
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