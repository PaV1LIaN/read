Часть 2 из 8:

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

Продолжу частью 3 из 8 следующим сообщением.