Продолжение:

notify('Columns2 создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create (columns2)'));
      }
    });
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

    const render = () => `
      <div class="galleryBuilder">
        <div class="field">
          <label>Колонки</label>
          <select id="gb_cols" class="input">
            <option value="2" ${columns===2?'selected':''}>2</option>
            <option value="3" ${columns===3?'selected':''}>3</option>
            <option value="4" ${columns===4?'selected':''}>4</option>
          </select>
        </div>

        <div style="margin-top:10px;">
          <button class="ui-btn ui-btn-light" id="gb_add">+ Добавить изображение</button>
        </div>

        <div id="gb_items">${renderItems()}</div>
      </div>
    `;

    BX.UI.Dialogs.MessageBox.show({
      title: mode === 'edit' ? ('Редактировать Gallery #' + blockId) : 'Новая Gallery',
      message: render(),
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb){
        columns = parseInt(document.getElementById('gb_cols')?.value || '3', 10);
        if (![2,3,4].includes(columns)) columns = 3;

        const collected = images.map((_, idx) => {
          const fileId = parseInt(document.querySelector(`[data-gallery-file="${idx}"]`)?.value || '0', 10) || 0;
          const alt = document.querySelector(`[data-gallery-alt="${idx}"]`)?.value || '';
          return { fileId, alt };
        }).filter(x => x.fileId > 0);

        if (!collected.length) { notify('Добавь хотя бы одно изображение'); return; }

        const payload = {
          columns,
          images: JSON.stringify(collected)
        };

        const call = (mode === 'edit')
          ? api('block.update', Object.assign({ id: blockId }, payload))
          : api('block.create', Object.assign({ pageId, type:'gallery' }, payload));

        call.then(res => {
          if (!res || res.ok !== true) {
            notify(mode === 'edit' ? 'Не удалось сохранить gallery' : 'Не удалось создать gallery');
            return;
          }
          notify(mode === 'edit' ? 'Сохранено' : 'Gallery создан');
          mb.close();
          loadBlocks();
        }).catch(() => notify(mode === 'edit' ? 'Ошибка block.update (gallery)' : 'Ошибка block.create (gallery)'));
      }
    });

    setTimeout(() => {
      const root = document.querySelector('.galleryBuilder');
      if (!root) return;

      const snapshot = () => {
        images = images.map((it, idx) => ({
          fileId: parseInt(document.querySelector(`[data-gallery-file="${idx}"]`)?.value || it.fileId || 0, 10) || 0,
          alt: document.querySelector(`[data-gallery-alt="${idx}"]`)?.value || it.alt || ''
        }));
        columns = parseInt(document.getElementById('gb_cols')?.value || String(columns), 10);
        if (![2,3,4].includes(columns)) columns = 3;
      };

      const rerender = () => {
        snapshot();
        root.innerHTML = render();
        bind();
      };

      const bind = () => {
        const addBtn = document.getElementById('gb_add');
        if (addBtn) addBtn.onclick = () => { snapshot(); images.push({fileId:0, alt:''}); rerender(); };

        root.querySelectorAll('[data-gallery-up]').forEach(btn => {
          btn.onclick = () => {
            snapshot();
            const i = parseInt(btn.getAttribute('data-gallery-up'), 10);
            if (i > 0) { [images[i-1], images[i]] = [images[i], images[i-1]]; rerender(); }
          };
        });

        root.querySelectorAll('[data-gallery-down]').forEach(btn => {
          btn.onclick = () => {
            snapshot();
            const i = parseInt(btn.getAttribute('data-gallery-down'), 10);
            if (i < images.length - 1) { [images[i+1], images[i]] = [images[i], images[i+1]]; rerender(); }
          };
        });

        root.querySelectorAll('[data-gallery-del]').forEach(btn => {
          btn.onclick = () => {
            snapshot();
            const i = parseInt(btn.getAttribute('data-gallery-del'), 10);
            images.splice(i, 1);
            if (!images.length) images.push({fileId:0, alt:''});
            rerender();
          };
        });

        root.querySelectorAll('select[data-gallery-file]').forEach(sel => {
          sel.onchange = () => {
            const idx = parseInt(sel.getAttribute('data-gallery-file'), 10);
            const fid = parseInt(sel.value || '0', 10);
            const box = root.querySelector(`[data-gallery-prev="${idx}"]`);
            if (!box) return;
            box.innerHTML = fid ? `<div class="imgPrev"><img src="${fileDownloadUrl(fid)}" alt=""></div>` : '';
          };
        });
      };

      bind();
    }, 0);
  }

  function addGalleryBlock() {
    openGalleryDialog('create', 0, null);
  }

  function addSpacerBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Spacer блок',
      message: `
        <div>
          <div class="field">
            <label>Высота (10..200 px)</label>
            <input id="sp_h" class="input" type="number" min="10" max="200" value="40" />
          </div>
          <div class="field">
            <label><input id="sp_line" type="checkbox" /> Рисовать линию</label>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb){
        const height = parseInt(document.getElementById('sp_h')?.value || '40', 10);
        const line = document.getElementById('sp_line')?.checked ? '1' : '0';

        api('block.create', { pageId, type:'spacer', height, line })
          .then(res => {
            if (!res || res.ok !== true) { notify('Не удалось создать spacer'); return; }
            notify('Spacer создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create (spacer)'));
      }
    });
  }

Шлю следующий кусок после openCardDialog(...), addCardBlock(), addCardsBlock() и edit... функций.