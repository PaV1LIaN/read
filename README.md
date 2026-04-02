window.SBEditor = window.SBEditor || {};

window.SBEditor.saveTemplateFromPage = async function () {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  const name = prompt('Название шаблона:');
  if (!name) return;

  try {
    const res = await api('template.createFromPage', {
      siteId: st.siteId,
      pageId: st.pageId,
      name: name.trim()
    });

    if (!res || res.ok !== true) {
      notify('Не удалось сохранить шаблон');
      return;
    }

    notify('Шаблон сохранён');
  } catch (e) {
    console.error(e);
    notify('Ошибка template.createFromPage');
  }
};

window.SBEditor.applyTemplateToPage = async function () {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  try {
    const listRes = await api('template.list', {});
    if (!listRes || listRes.ok !== true) {
      notify('Не удалось загрузить шаблоны');
      return;
    }

    const templates = Array.isArray(listRes.templates) ? listRes.templates : [];
    if (!templates.length) {
      notify('Шаблонов пока нет');
      return;
    }

    const options = templates.map(t => {
      return `<option value="${parseInt(t.id, 10)}">${BX.util.htmlspecialchars(String(t.name || ('Template #' + t.id)))}</option>`;
    }).join('');

    BX.UI.Dialogs.MessageBox.show({
      title: 'Применить шаблон',
      message: `
        <div>
          <div class="field">
            <label>Шаблон</label>
            <select id="tpl_apply_id" class="input">${options}</select>
          </div>
          <div class="field">
            <label>Режим</label>
            <select id="tpl_apply_mode" class="input">
              <option value="append">Добавить к текущим блокам</option>
              <option value="replace">Заменить все текущие блоки</option>
            </select>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: async function (mbox) {
        const templateId = parseInt(document.getElementById('tpl_apply_id')?.value || '0', 10);
        const mode = String(document.getElementById('tpl_apply_mode')?.value || 'append');

        if (!templateId) {
          notify('Шаблон не выбран');
          return;
        }

        try {
          const res = await api('template.applyToPage', {
            siteId: st.siteId,
            pageId: st.pageId,
            templateId,
            mode
          });

          if (!res || res.ok !== true) {
            notify('Не удалось применить шаблон');
            return;
          }

          notify('Шаблон применён');
          mbox.close();

          if (typeof window.SBEditor.loadBlocks === 'function') {
            window.SBEditor.loadBlocks();
          }
        } catch (e) {
          console.error(e);
          notify('Ошибка template.applyToPage');
        }
      }
    });
  } catch (e) {
    console.error(e);
    notify('Ошибка template.list');
  }
};

window.SBEditor.openSectionsLibrary = function () {
  const st = window.SBEditor.getState();
  const notify = window.SBEditor.notify;

  const presets = [
    {
      key: 'hero',
      title: 'Hero',
      text: 'Секция с крупным заголовком, текстом и кнопкой',
      create: async function () {
        await window.SBEditor.addSectionBlockWithPreset('hero');

        const listRes = await window.SBEditor.api('block.list', { pageId: st.pageId });
        const blocks = Array.isArray(listRes?.blocks) ? listRes.blocks.slice() : [];
        const sections = blocks.filter(b => String(b.type || '') === 'section').sort((a, b) => parseInt(b.id, 10) - parseInt(a.id, 10));
        const section = sections[0];
        if (!section) return;

        await window.SBEditor.quickAddHeadingAfterSection(section.id);
        await window.SBEditor.quickAddTextAfterSection(section.id);
        await window.SBEditor.quickAddButtonAfterSection(section.id);
      }
    },
    {
      key: 'cards',
      title: 'Cards section',
      text: 'Секция с карточками преимуществ',
      create: async function () {
        await window.SBEditor.addSectionBlockWithPreset('light');

        const listRes = await window.SBEditor.api('block.list', { pageId: st.pageId });
        const blocks = Array.isArray(listRes?.blocks) ? listRes.blocks.slice() : [];
        const sections = blocks.filter(b => String(b.type || '') === 'section').sort((a, b) => parseInt(b.id, 10) - parseInt(a.id, 10));
        const section = sections[0];
        if (!section) return;

        await window.SBEditor.quickAddHeadingAfterSection(section.id);
        await window.SBEditor.quickAddCardsAfterSection(section.id);
      }
    },
    {
      key: 'cta',
      title: 'CTA',
      text: 'Небольшая акцентная секция с кнопкой',
      create: async function () {
        await window.SBEditor.addSectionBlockWithPreset('accent');

        const listRes = await window.SBEditor.api('block.list', { pageId: st.pageId });
        const blocks = Array.isArray(listRes?.blocks) ? listRes.blocks.slice() : [];
        const sections = blocks.filter(b => String(b.type || '') === 'section').sort((a, b) => parseInt(b.id, 10) - parseInt(a.id, 10));
        const section = sections[0];
        if (!section) return;

        await window.SBEditor.quickAddHeadingAfterSection(section.id);
        await window.SBEditor.quickAddButtonAfterSection(section.id);
      }
    }
  ];

  BX.UI.Dialogs.MessageBox.show({
    title: 'Библиотека секций',
    message: `
      <div style="display:grid;gap:12px;">
        ${presets.map(p => `
          <div class="subCard" style="padding:12px;">
            <div style="font-weight:700;font-size:15px;">${BX.util.htmlspecialchars(p.title)}</div>
            <div class="muted" style="margin-top:6px;">${BX.util.htmlspecialchars(p.text)}</div>
            <div style="margin-top:10px;">
              <button class="ui-btn ui-btn-primary ui-btn-xs" data-sections-lib-key="${BX.util.htmlspecialchars(p.key)}">Создать</button>
            </div>
          </div>
        `).join('')}
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.CANCEL,
    onCancel: function () {}
  });

  setTimeout(() => {
    document.querySelectorAll('[data-sections-lib-key]').forEach(btn => {
      btn.addEventListener('click', async () => {
        const key = btn.getAttribute('data-sections-lib-key');
        const preset = presets.find(x => x.key === key);
        if (!preset) return;

        try {
          await preset.create();
          notify('Секция создана');

          if (typeof window.SBEditor.loadBlocks === 'function') {
            window.SBEditor.loadBlocks();
          }
        } catch (e) {
          console.error(e);
          notify('Ошибка создания секции');
        }
      });
    });
  }, 0);
};

window.SBEditor.openCardsBuilderDialog = function (opts = {}) {
  const title = String(opts.title || 'Карточки');
  const initialItems = Array.isArray(opts.items) ? opts.items.slice() : [];
  const initialColumns = parseInt(opts.columns || 3, 10);
  const onSubmit = typeof opts.onSubmit === 'function' ? opts.onSubmit : null;

  const rowsHtml = (initialItems.length ? initialItems : [
    { title: '', text: '', imageFileId: 0, buttonText: '', buttonUrl: '' }
  ]).map((item, idx) => {
    const clean = window.SBEditor.cardsNormalizeItem(item);

    return `
      <div class="cardsBuilderRow" data-cards-row="${idx}" style="border:1px solid #e5e7eb;border-radius:12px;padding:12px;margin-top:10px;">
        <div class="field">
          <label>Заголовок</label>
          <input class="input" data-role="title" value="${BX.util.htmlspecialchars(clean.title)}">
        </div>

        <div class="field">
          <label>Текст</label>
          <textarea class="input" rows="4" data-role="text">${BX.util.htmlspecialchars(clean.text)}</textarea>
        </div>

        <div class="field">
          <label>Image fileId</label>
          <input class="input" data-role="imageFileId" type="number" min="0" value="${clean.imageFileId}">
        </div>

        <div class="field">
          <label>Текст кнопки</label>
          <input class="input" data-role="buttonText" value="${BX.util.htmlspecialchars(clean.buttonText)}">
        </div>

        <div class="field">
          <label>URL кнопки</label>
          <input class="input" data-role="buttonUrl" value="${BX.util.htmlspecialchars(clean.buttonUrl)}">
        </div>

        <div style="margin-top:8px;">
          <button type="button" class="ui-btn ui-btn-light ui-btn-xs" data-remove-cards-row="${idx}">Удалить карточку</button>
        </div>
      </div>
    `;
  }).join('');

buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: async function(mb){
        cols = parseInt(document.getElementById('cb_cols')?.value || '3', 10);
        if (![2,3,4].includes(cols)) cols = 3;

        const nextItems = [];
        const rows = Array.from(document.querySelectorAll('#cb_items .item'));
        rows.forEach((row, idx) => {
          const title = (row.querySelector(`[data-card-title="${idx}"]`)?.value || '').trim();
          const text = row.querySelector(`[data-card-text="${idx}"]`)?.value || '';
          const imageFileId = parseInt(row.querySelector(`[data-card-img="${idx}"]`)?.value || '0', 10) || 0;
          const buttonText = (row.querySelector(`[data-card-btntext="${idx}"]`)?.value || '').trim();
          const buttonUrl = (row.querySelector(`[data-card-btnurl="${idx}"]`)?.value || '').trim();

          const clean = cardsNormalizeItem({ title, text, imageFileId, buttonText, buttonUrl });
          if (clean.title) nextItems.push(clean);
        });

        if (!nextItems.length) { notify('Добавь хотя бы одну карточку с заголовком'); return; }

        try {
          const payload = {
            columns: cols,
            items: JSON.stringify(nextItems)
          };

          let r;
          if (mode === 'edit') {
            r = await api('block.update', { id: blockId, ...payload });
          } else {
            r = await api('block.create', { pageId, type: 'cards', ...payload });
          }

          if (!r || r.ok !== true) {
            notify(mode === 'edit' ? 'Не удалось сохранить cards' : 'Не удалось создать cards');
            return;
          }

          notify(mode === 'edit' ? 'Cards сохранён' : 'Cards блок создан');
          mb.close();
          loadBlocks();
        } catch (e) {
          notify(mode === 'edit' ? 'Ошибка block.update (cards)' : 'Ошибка block.create (cards)');
        }
      }
    });

    setTimeout(() => {
      const root = document.querySelector('.cardsBuilder');
      if (!root) return;

      const rerender = () => {
        const wrap = document.querySelector('#cb_items');
        if (!wrap) return;
        wrap.innerHTML = cardsRenderBuilderItems(items, files);

        bindRows();
      };

      const bindRows = () => {
        document.querySelectorAll('[data-card-del]').forEach(btn => {
          btn.onclick = () => {
            const idx = parseInt(btn.getAttribute('data-card-del') || '-1', 10);
            if (idx < 0) return;
            items.splice(idx, 1);
            if (!items.length) items.push(cardsNormalizeItem({title:'', text:''}));
            rerender();
          };
        });

        document.querySelectorAll('[data-card-up]').forEach(btn => {
          btn.onclick = () => {
            const idx = parseInt(btn.getAttribute('data-card-up') || '-1', 10);
            if (idx <= 0) return;
            const tmp = items[idx - 1];
            items[idx - 1] = items[idx];
            items[idx] = tmp;
            rerender();
          };
        });

        document.querySelectorAll('[data-card-down]').forEach(btn => {
          btn.onclick = () => {
            const idx = parseInt(btn.getAttribute('data-card-down') || '-1', 10);
            if (idx < 0 || idx >= items.length - 1) return;
            const tmp = items[idx + 1];
            items[idx + 1] = items[idx];
            items[idx] = tmp;
            rerender();
          };
        });

        document.querySelectorAll('[data-card-title]').forEach(inp => {
          inp.oninput = () => {
            const idx = parseInt(inp.getAttribute('data-card-title') || '-1', 10);
            if (idx >= 0 && items[idx]) items[idx].title = inp.value || '';
          };
        });

        document.querySelectorAll('[data-card-text]').forEach(inp => {
          inp.oninput = () => {
            const idx = parseInt(inp.getAttribute('data-card-text') || '-1', 10);
            if (idx >= 0 && items[idx]) items[idx].text = inp.value || '';
          };
        });

        document.querySelectorAll('[data-card-btntext]').forEach(inp => {
          inp.oninput = () => {
            const idx = parseInt(inp.getAttribute('data-card-btntext') || '-1', 10);
            if (idx >= 0 && items[idx]) items[idx].buttonText = inp.value || '';
          };
        });

        document.querySelectorAll('[data-card-btnurl]').forEach(inp => {
          inp.oninput = () => {
            const idx = parseInt(inp.getAttribute('data-card-btnurl') || '-1', 10);
            if (idx >= 0 && items[idx]) items[idx].buttonUrl = inp.value || '';
          };
        });

        document.querySelectorAll('[data-card-img]').forEach(sel => {
          sel.onchange = () => {
            const idx = parseInt(sel.getAttribute('data-card-img') || '-1', 10);
            if (idx >= 0 && items[idx]) {
              items[idx].imageFileId = parseInt(sel.value || '0', 10) || 0;
              const prev = document.querySelector(`[data-card-img-prev="${idx}"]`);
              if (prev) {
                prev.innerHTML = items[idx].imageFileId
                  ? `<div class="imgPrev"><img src="${fileDownloadUrl(items[idx].imageFileId)}" alt=""></div>`
                  : '';
              }
            }
          };
        });
      };

      document.getElementById('cb_add')?.addEventListener('click', () => {
        items.push(cardsNormalizeItem({title:'', text:''}));
        rerender();
      });

      document.getElementById('cb_cols')?.addEventListener('change', (e) => {
        cols = parseInt(e.target.value || '3', 10);
        if (![2,3,4].includes(cols)) cols = 3;
      });

      bindRows();
    }, 0);
  }

  async function openGalleryDialog(mode, blockId, currentContent) {
    let files = [];
    try { files = await getFilesForSite(); } catch(e) { files = []; }

    let columns = currentContent?.columns ? parseInt(currentContent.columns, 10) : 3;
    if (![2,3,4].includes(columns)) columns = 3;

    let images = Array.isArray(currentContent?.images) ? currentContent.images.map(x => ({
      fileId: parseInt(x.fileId || 0, 10) || 0,
      alt: x.alt || ''
    })) : [{fileId:0, alt:''}];

    const fileOptions = (selectedId) => {
      const opts = ['<option value="0">— выбрать файл —</option>'];
      files.forEach(f => {
        const s = (parseInt(f.id,10) === selectedId) ? 'selected' : '';
        opts.push(`<option value="${f.id}" ${s}>${BX.util.htmlspecialchars(f.name)} (id ${f.id})</option>`);
      });
      return opts.join('');
    };

    const renderItems = () => images.map((it, idx) => `
      <div class="item" data-gi="${idx}">
        <div class="itemHead">
          <div><b>Изображение ${idx + 1}</b></div>
          <div class="miniBtns">
            <button class="ui-btn ui-btn-light ui-btn-xs" data-gallery-up="${idx}">↑</button>
            <button class="ui-btn ui-btn-light ui-btn-xs" data-gallery-down="${idx}">↓</button>
            <button class="ui-btn ui-btn-danger ui-btn-xs" data-gallery-del="${idx}">Удалить</button>
          </div>
        </div>

        <div class="grid2" style="margin-top:10px;">
          <div>
            <div class="field">
              <label>Файл</label>
              <select class="input" data-gallery-file="${idx}">
                ${fileOptions(parseInt(it.fileId || 0, 10))}
              </select>
            </div>

            <div data-gallery-prev="${idx}">
              ${parseInt(it.fileId || 0, 10) ? `<div class="imgPrev"><img src="${fileDownloadUrl(parseInt(it.fileId || 0, 10))}" alt=""></div>` : ''}
            </div>
          </div>

          <div>
            <div class="field">
              <label>Alt</label>
              <input class="input" data-gallery-alt="${idx}" value="${BX.util.htmlspecialchars(it.alt || '')}">
            </div>
          </div>
        </div>
      </div>
    `).join('');
