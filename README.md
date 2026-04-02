Часть 3 из 8:

<button class="ui-btn ui-btn-danger ui-btn-xs" data-tpl-delete="${t.id}">Удалить</button>
            </div>
          </div>
        `;
      }).join('') || '<div class="muted">Ничего не найдено</div>';

      return `
        <div class="secSearchWrap">
          <input class="input" id="secSearchInput" placeholder="Поиск шаблона по названию..." value="${BX.util.htmlspecialchars(query)}">
        </div>
        <div class="secGrid">${cards}</div>
      `;
    };

    const mb = BX.UI.Dialogs.MessageBox.show({
      title: 'Библиотека секций / шаблонов',
      message: `<div id="${containerId}">${render('')}</div>`,
      buttons: BX.UI.Dialogs.MessageBoxButtons.CANCEL,
      popupOptions: { width: 980, closeIcon: true, overlay: true }
    });

    function bindHandlers(root) {
      const search = root.querySelector('#secSearchInput');
      if (search) {
        search.oninput = () => {
          root.innerHTML = render(search.value || '');
          bindHandlers(root);
        };
      }

      root.querySelectorAll('[data-tpl-apply]').forEach(btn => {
        btn.onclick = async () => {
          const templateId = parseInt(btn.getAttribute('data-tpl-apply'), 10);
          const mode = btn.getAttribute('data-mode') || 'append';
          try {
            const r = await api('template.applyToPage', {
              siteId,
              pageId,
              templateId,
              mode
            });
            if (!r || r.ok !== true) { notify('Не удалось применить шаблон'); return; }
            notify(mode === 'replace' ? 'Страница заменена шаблоном' : 'Шаблон вставлен');
            mb.close();
            loadBlocks();
          } catch (e) {
            notify('Ошибка template.applyToPage');
          }
        };
      });

      root.querySelectorAll('[data-tpl-rename]').forEach(btn => {
        btn.onclick = async () => {
          const id = parseInt(btn.getAttribute('data-tpl-rename'), 10);
          const cur = templates.find(x => parseInt(x.id, 10) === id);
          const name = prompt('Новое имя шаблона:', cur?.name || '');
          if (!name) return;

          try {
            const r = await api('template.rename', { id, name });
            if (!r || r.ok !== true) { notify('Не удалось переименовать шаблон'); return; }
            notify('Шаблон переименован');

            const fresh = await api('template.list', {});
            templates = (fresh && fresh.ok === true && Array.isArray(fresh.templates)) ? fresh.templates : templates;
            const rootEl = document.getElementById(containerId);
            if (rootEl) {
              const q = rootEl.querySelector('#secSearchInput')?.value || '';
              rootEl.innerHTML = render(q);
              bindHandlers(rootEl);
            }
          } catch (e) {
            notify('Ошибка template.rename');
          }
        };
      });

      root.querySelectorAll('[data-tpl-delete]').forEach(btn => {
        btn.onclick = async () => {
          const id = parseInt(btn.getAttribute('data-tpl-delete'), 10);
          if (!confirm('Удалить шаблон #' + id + '?')) return;

          try {
            const r = await api('template.delete', { id });
            if (!r || r.ok !== true) { notify('Не удалось удалить шаблон'); return; }
            notify('Шаблон удалён');

            const fresh = await api('template.list', {});
            templates = (fresh && fresh.ok === true && Array.isArray(fresh.templates)) ? fresh.templates : templates;
            const rootEl = document.getElementById(containerId);
            if (rootEl) {
              const q = rootEl.querySelector('#secSearchInput')?.value || '';
              rootEl.innerHTML = render(q);
              bindHandlers(rootEl);
            }
          } catch (e) {
            notify('Ошибка template.delete');
          }
        };
      });
    }

    setTimeout(() => {
      const root = document.getElementById(containerId);
      if (root) bindHandlers(root);
    }, 0);
  }

  function addTextBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Text',
      message: `
        <div class="field">
          <label>Текст</label>
          <textarea id="new_text_value" class="input" rows="10" placeholder="Введите текст блока"></textarea>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = document.getElementById('new_text_value')?.value || '';
        api('block.create', { pageId, type: 'text', text })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать text'); return; }
            notify('Text блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });
  }

  async function addImageBlock() {
    let files = [];
    try { files = await getFilesForSite(); }
    catch (e) { notify('Не удалось загрузить файлы сайта'); return; }

    const opts = ['<option value="0">— выбрать файл —</option>']
      .concat(files.map(f => `<option value="${f.id}">${BX.util.htmlspecialchars(f.name)} (#${f.id})</option>`))
      .join('');

    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Image',
      message: `
        <div class="field">
          <label>Файл</label>
          <select id="new_image_file" class="input">${opts}</select>
        </div>
        <div class="field">
          <label>Alt</label>
          <input id="new_image_alt" class="input" placeholder="Описание изображения" />
        </div>
        <div id="new_image_preview_wrap" style="margin-top:10px;"></div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const fileId = parseInt(document.getElementById('new_image_file')?.value || '0', 10);
        const alt = document.getElementById('new_image_alt')?.value || '';

        if (!fileId) { notify('Выбери файл'); return; }

        api('block.create', { pageId, type: 'image', fileId, alt })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать image'); return; }
            notify('Image блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });

    setTimeout(() => {
      const sel = document.getElementById('new_image_file');
      const wrap = document.getElementById('new_image_preview_wrap');
      if (!sel || !wrap) return;

      const renderPrev = () => {
        const fid = parseInt(sel.value || '0', 10);
        wrap.innerHTML = fid
          ? `<div class="imgPrev"><img src="${fileDownloadUrl(fid)}" alt=""></div>`
          : '';
      };

      sel.addEventListener('change', renderPrev);
      renderPrev();
    }, 0);
  }

  function addButtonBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Button',
      message: `
        <div class="field">
          <label>Текст кнопки</label>
          <input id="new_btn_text" class="input" value="Кнопка" />
        </div>
        <div class="field">
          <label>URL</label>
          <input id="new_btn_url" class="input" value="/" />
        </div>
        <div class="field">
          <label>Вариант</label>
          <select id="new_btn_variant" class="input">
            <option value="primary">primary</option>
            <option value="secondary">secondary</option>
          </select>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = document.getElementById('new_btn_text')?.value || '';
        const url = document.getElementById('new_btn_url')?.value || '';
        const variant = document.getElementById('new_btn_variant')?.value || 'primary';

        api('block.create', { pageId, type: 'button', text, url, variant })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать button'); return; }
            notify('Button блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });
  }

  function addHeadingBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Heading',
      message: `
        <div class="field">
          <label>Текст</label>
          <input id="new_heading_text" class="input" value="Новый заголовок" />
        </div>
        <div class="field">
          <label>Уровень</label>
          <select id="new_heading_level" class="input">
            <option value="h1">h1</option>
            <option value="h2" selected>h2</option>
            <option value="h3">h3</option>
          </select>
        </div>
        <div class="field">
          <label>Выравнивание</label>
          <select id="new_heading_align" class="input">
            <option value="left" selected>left</option>
            <option value="center">center</option>
            <option value="right">right</option>
          </select>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = document.getElementById('new_heading_text')?.value || '';
        const level = document.getElementById('new_heading_level')?.value || 'h2';
        const align = document.getElementById('new_heading_align')?.value || 'left';

Пришлю часть 4 следующим сообщением.