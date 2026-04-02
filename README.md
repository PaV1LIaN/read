Идём дальше.

Сейчас дам editor.blocks.js в виде готового файла. Он опирается на:

window.SBEditor.getState()

window.SBEditor.fileDownloadUrl

window.SBEditor.btnClass

window.SBEditor.headingTag

window.SBEditor.headingAlign

window.SBEditor.colsGridTemplate

window.SBEditor.galleryTemplate

window.SBEditor.cardsNormalizeItem

window.SBEditor.editSectionBlock


Создай файл:

/local/sitebuilder/assets/js/editor.blocks.js

и вставь туда:

window.SBEditor = window.SBEditor || {};

window.SBEditor.buildBlockShell = function (
  id,
  type,
  sort,
  bodyHtml,
  buttonsHtml,
  extraMetaHtml = '',
  extraClass = '',
  sectionMarkHtml = ''
) {
  const st = window.SBEditor.getState();
  const collapsed = st.collapsedBlocks instanceof Set && st.collapsedBlocks.has(id);
  const typeLabel = String(type || '').toUpperCase();

  return `
    <div class="block ${collapsed ? 'blockCollapsed' : ''} ${extraClass}" data-type="${BX.util.htmlspecialchars(type)}" data-block-id="${id}">
      <div class="row">
        <div style="display:flex;align-items:center;gap:8px;flex-wrap:wrap;min-width:0;">
          <span class="dragHandle" data-drag-handle title="Перетащить">⋮⋮</span>
          <b>#${id}</b>
          <span class="badge">${BX.util.htmlspecialchars(typeLabel)}</span>
          <span class="muted">(sort ${sort})</span>
          ${extraMetaHtml || ''}
        </div>

        <div class="btns">
          <button class="ui-btn ui-btn-light ui-btn-xs" data-toggle-block-id="${id}">
            ${collapsed ? 'Развернуть' : 'Свернуть'}
          </button>
          ${buttonsHtml || ''}
        </div>
      </div>

      <div class="blockBody">
        ${sectionMarkHtml || ''}
        ${bodyHtml || ''}
      </div>
    </div>
  `;
};

window.SBEditor.renderBlocks = function (blocks) {
  const st = window.SBEditor.getState();
  const blocksBox = st.blocksBox;
  const blockSearch = st.blockSearch;

  if (!blocksBox) return;

  if (!Array.isArray(blocks) || blocks.length === 0) {
    blocksBox.innerHTML = '<div class="muted">Нет блоков</div>';
    return;
  }

  const q = (blockSearch?.value || '').trim().toLowerCase();

  const filteredBlocks = blocks.filter(b => {
    if (!q) return true;
    const type = String(b.type || '').toLowerCase();
    const raw = JSON.stringify(b.content || {}).toLowerCase();
    return type.includes(q) || raw.includes(q) || String(b.id || '').includes(q);
  });

  if (!filteredBlocks.length) {
    blocksBox.innerHTML = '<div class="muted">Ничего не найдено</div>';
    return;
  }

  let currentSectionId = null;

  function wrapBlockMeta() {
    if (!currentSectionId) {
      return {
        extraClass: '',
        sectionMarkHtml: ''
      };
    }

    return {
      extraClass: 'blockInSection',
      sectionMarkHtml: `<div class="blockSectionDivider">inside section #${currentSectionId}</div>`
    };
  }

  blocksBox.innerHTML = filteredBlocks.map(b => {
    const type = String(b.type || '');
    const sort = parseInt(b.sort || 0, 10);
    const id = parseInt(b.id || 0, 10);
    const c = (b.content && typeof b.content === 'object') ? b.content : {};
    const commonBtns = `
      <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
      <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
    `;

    if (type === 'section') {
      currentSectionId = id;

      const boxed = !!c.boxed;
      const background = c.background || '#FFFFFF';
      const paddingTop = parseInt(c.paddingTop || 32, 10);
      const paddingBottom = parseInt(c.paddingBottom || 32, 10);
      const border = !!c.border;
      const radius = parseInt(c.radius || 0, 10);

      return window.SBEditor.buildBlockShell(
        id,
        type,
        sort,
        `
          <div class="blockSectionGrid">
            <div class="blockSectionItem">
              <div class="blockSectionLabel">Контейнер</div>
              <div class="blockSectionValue">${boxed ? 'Boxed' : 'Full width'}</div>
            </div>

            <div class="blockSectionItem">
              <div class="blockSectionLabel">Фон</div>
              <div class="blockSectionValue">
                <span style="display:inline-block;width:12px;height:12px;border-radius:3px;background:${BX.util.htmlspecialchars(background)};border:1px solid #cbd5e1;vertical-align:-1px;margin-right:6px;"></span>
                ${BX.util.htmlspecialchars(background)}
              </div>
            </div>

            <div class="blockSectionItem">
              <div class="blockSectionLabel">Отступы</div>
              <div class="blockSectionValue">top ${paddingTop}px / bottom ${paddingBottom}px</div>
            </div>

            <div class="blockSectionItem">
              <div class="blockSectionLabel">Граница</div>
              <div class="blockSectionValue">${border ? 'Да' : 'Нет'}</div>
            </div>

            <div class="blockSectionItem">
              <div class="blockSectionLabel">Скругление</div>
              <div class="blockSectionValue">${radius}px</div>
            </div>
          </div>

          <div class="btns" style="margin-top:10px;">
            <button class="ui-btn ui-btn-success ui-btn-xs" data-add-heading-after-section-id="${id}">+ Heading</button>
            <button class="ui-btn ui-btn-success ui-btn-xs" data-add-text-after-section-id="${id}">+ Text</button>
            <button class="ui-btn ui-btn-success ui-btn-xs" data-add-button-after-section-id="${id}">+ Button</button>
            <button class="ui-btn ui-btn-success ui-btn-xs" data-add-cards-after-section-id="${id}">+ Cards</button>
          </div>
        `,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-section-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-dup-block-id="${id}">Дублировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        '',
        'blockSection'
      );
    }

    if (type === 'text') {
      const wrap = wrapBlockMeta();
      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `<pre>${BX.util.htmlspecialchars(String(c.text || ''))}</pre>`,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        '',
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'image') {
      const wrap = wrapBlockMeta();
      const fileId = parseInt(c.fileId || 0, 10);
      const alt = String(c.alt || '');
      const imgHtml = fileId > 0
        ? `<img src="${window.SBEditor.fileDownloadUrl(st.siteId, fileId)}" alt="${BX.util.htmlspecialchars(alt)}" style="max-width:100%;border-radius:10px;">`
        : `<div class="muted">Изображение не выбрано</div>`;

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        imgHtml,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-image-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        fileId > 0 ? `<span class="muted">fileId ${fileId}</span>` : '',
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'button') {
      const wrap = wrapBlockMeta();
      const text = String(c.text || '');
      const url = String(c.url || '');
      const variant = String(c.variant || 'primary');

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `
          <div style="display:flex;gap:10px;align-items:center;flex-wrap:wrap;">
            <button class="ui-btn ${window.SBEditor.btnClass(variant)}">${BX.util.htmlspecialchars(text || 'Кнопка')}</button>
            <span class="muted">${BX.util.htmlspecialchars(url)}</span>
          </div>
        `,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-button-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        '',
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'heading') {
      const wrap = wrapBlockMeta();
      const text = String(c.text || '');
      const level = window.SBEditor.headingTag(String(c.level || 'h2'));
      const align = window.SBEditor.headingAlign(String(c.align || 'left'));

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `<${level} style="margin:0;text-align:${align};">${BX.util.htmlspecialchars(text)}</${level}>`,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-heading-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        '',
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'columns2') {
      const wrap = wrapBlockMeta();
      const left = String(c.left || '');
      const right = String(c.right || '');
      const ratio = String(c.ratio || '50-50');
      const tpl = window.SBEditor.colsGridTemplate(ratio);

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `
          <div style="display:grid;grid-template-columns:${tpl};gap:12px;">
            <div class="subCard"><pre>${BX.util.htmlspecialchars(left)}</pre></div>
            <div class="subCard"><pre>${BX.util.htmlspecialchars(right)}</pre></div>
          </div>
        `,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cols2-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        '',
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'gallery') {
      const wrap = wrapBlockMeta();
      const columns = parseInt(c.columns || 3, 10);
      const images = Array.isArray(c.images) ? c.images : [];
      const tpl = window.SBEditor.galleryTemplate(columns);

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `
          <div style="display:grid;grid-template-columns:${tpl};gap:12px;">
            ${images.map(img => {
              const fid = parseInt(img.fileId || 0, 10);
              if (fid <= 0) return '';
              return `<img src="${window.SBEditor.fileDownloadUrl(st.siteId, fid)}" alt="${BX.util.htmlspecialchars(String(img.alt || ''))}" style="width:100%;border-radius:10px;">`;
            }).join('')}
          </div>
        `,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-gallery-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        `<span class="muted">${images.length} изображ.</span>`,
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'spacer') {
      const wrap = wrapBlockMeta();
      const height = parseInt(c.height || 40, 10);
      const line = !!c.line;

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `
          <div style="height:${height}px;position:relative;background:#f8fafc;border-radius:8px;">
            ${line ? '<div style="position:absolute;left:0;right:0;top:50%;height:1px;background:#cbd5e1;"></div>' : ''}
          </div>
        `,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-spacer-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        `<span class="muted">${height}px</span>`,
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'card') {
      const wrap = wrapBlockMeta();
      const title = String(c.title || '');
      const text = String(c.text || '');
      const imageFileId = parseInt(c.imageFileId || 0, 10);
      const buttonText = String(c.buttonText || '');
      const buttonUrl = String(c.buttonUrl || '');

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `
          <div class="subCard">
            <div style="font-weight:700;">${BX.util.htmlspecialchars(title)}</div>
            ${text ? `<pre style="margin-top:8px;">${BX.util.htmlspecialchars(text)}</pre>` : ''}
            ${imageFileId > 0 ? `<img src="${window.SBEditor.fileDownloadUrl(st.siteId, imageFileId)}" style="max-width:100%;margin-top:10px;border-radius:10px;">` : ''}
            ${buttonUrl ? `<div style="margin-top:10px;"><button class="ui-btn ui-btn-light">${BX.util.htmlspecialchars(buttonText || 'Открыть')}</button> <span class="muted">${BX.util.htmlspecialchars(buttonUrl)}</span></div>` : ''}
          </div>
        `,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-card-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        '',
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    if (type === 'cards') {
      const wrap = wrapBlockMeta();
      const columns = parseInt(c.columns || 3, 10);
      const items = Array.isArray(c.items) ? c.items.map(window.SBEditor.cardsNormalizeItem) : [];
      const tpl = window.SBEditor.galleryTemplate(columns);

      return window.SBEditor.buildBlockShell(
        id, type, sort,
        `
          <div style="display:grid;grid-template-columns:${tpl};gap:12px;">
            ${items.map(it => `
              <div class="subCard">
                <div style="font-weight:700;">${BX.util.htmlspecialchars(it.title)}</div>
                ${it.text ? `<pre style="margin-top:8px;">${BX.util.htmlspecialchars(it.text)}</pre>` : ''}
                ${it.imageFileId > 0 ? `<img src="${window.SBEditor.fileDownloadUrl(st.siteId, it.imageFileId)}" style="max-width:100%;margin-top:10px;border-radius:10px;">` : ''}
                ${it.buttonUrl ? `<div style="margin-top:10px;"><button class="ui-btn ui-btn-light">${BX.util.htmlspecialchars(it.buttonText || 'Открыть')}</button></div>` : ''}
              </div>
            `).join('')}
          </div>
        `,
        `
          ${commonBtns}
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-cards-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        `,
        `<span class="muted">${items.length} шт.</span>`,
        wrap.extraClass,
        wrap.sectionMarkHtml
      );
    }

    const wrap = wrapBlockMeta();
    return window.SBEditor.buildBlockShell(
      id, type, sort,
      `<div class="muted">Неизвестный тип блока: ${BX.util.htmlspecialchars(type)}</div>`,
      `
        ${commonBtns}
        <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
      `,
      '',
      wrap.extraClass,
      wrap.sectionMarkHtml
    );
  }).join('');
};

Следующий файл — editor.dialogs.js.
Он будет самым большим, потому что туда уйдут все add...Block() и edit...Block() для обычных блоков.

Напиши дальше, и я дам его следующим.