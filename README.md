Вот вторая часть editor.dialogs.js, сразу после первой:

setTimeout(() => {
      const root = document.querySelector('.cardsBuilder');
      if (!root) return;

      const snapshot = () => {
        items = items.map((it, idx) => ({
          title: (document.querySelector(`[data-card-title="${idx}"]`)?.value || it.title || ''),
          text: (document.querySelector(`[data-card-text="${idx}"]`)?.value || it.text || ''),
          imageFileId: parseInt(document.querySelector(`[data-card-img="${idx}"]`)?.value || it.imageFileId || 0, 10) || 0,
          buttonText: (document.querySelector(`[data-card-btntext="${idx}"]`)?.value || it.buttonText || ''),
          buttonUrl: (document.querySelector(`[data-card-btnurl="${idx}"]`)?.value || it.buttonUrl || ''),
        }));
        cols = parseInt(document.getElementById('cb_cols')?.value || String(cols), 10);
        if (![2,3,4].includes(cols)) cols = 3;
      };

      const rerender = () => {
        snapshot();
        root.innerHTML = render();
        bind();
      };

      const bind = () => {
        const addBtn = document.getElementById('cb_add');
        if (addBtn) addBtn.onclick = () => { snapshot(); items.push(cardsNormalizeItem({ title: 'Новая карточка' })); rerender(); };

        root.querySelectorAll('[data-card-up]').forEach(btn => {
          btn.onclick = () => { snapshot(); const i = parseInt(btn.getAttribute('data-card-up'), 10);
            if (i > 0) { [items[i-1], items[i]] = [items[i], items[i-1]]; rerender(); }
          };
        });
        root.querySelectorAll('[data-card-down]').forEach(btn => {
          btn.onclick = () => { snapshot(); const i = parseInt(btn.getAttribute('data-card-down'), 10);
            if (i < items.length - 1) { [items[i+1], items[i]] = [items[i], items[i+1]]; rerender(); }
          };
        });
        root.querySelectorAll('[data-card-del]').forEach(btn => {
          btn.onclick = () => { snapshot(); const i = parseInt(btn.getAttribute('data-card-del'), 10);
            items.splice(i, 1); if (!items.length) items.push(cardsNormalizeItem({ title: 'Новая карточка' })); rerender();
          };
        });

        root.querySelectorAll('select[data-card-img]').forEach(sel => {
          sel.onchange = () => {
            const idx = parseInt(sel.getAttribute('data-card-img'), 10);
            const fid = parseInt(sel.value || '0', 10);
            const box = root.querySelector(`[data-card-img-prev="${idx}"]`);
            if (!box) return;
            box.innerHTML = fid ? `<div class="imgPrev"><img src="${fileDownloadUrl(fid)}" alt=""></div>` : '';
          };
        });
      };

      bind();
    }, 0);
  }


  async function openSectionsLibrary() {
    let res;
    try { res = await api('template.list', {}); }
    catch (e) { notify('Ошибка template.list'); return; }

    if (!res || res.ok !== true) { notify('Не удалось получить шаблоны'); return; }

    let templates = res.templates || [];
    if (!templates.length) { notify('Шаблонов нет. Сначала сохрани страницу как шаблон.'); return; }

    const containerId = 'sb_sections_root_' + Date.now();

    const render = (q) => {
      const query = (q || '').trim().toLowerCase();
      const filtered = templates.filter(t => ((t.name || '') + '').toLowerCase().includes(query));

      const cards = filtered.map(t => {
        const blocksCount = Array.isArray(t.blocks) ? t.blocks.length : 0;
        const createdAt = (t.createdAt || '').replace('T',' ').replace('Z','');

        return `
          <div class="secCard">
            <div class="secTitle">${BX.util.htmlspecialchars(t.name || ('Template #' + t.id))}</div>
            <div class="secMeta">id: ${t.id} • блоков: ${blocksCount} • создан: ${BX.util.htmlspecialchars(createdAt)}</div>

            <div class="secBtns">
              <button class="ui-btn ui-btn-primary ui-btn-xs" data-tpl-apply="${t.id}" data-mode="append">Вставить</button>
              <button class="ui-btn ui-btn-light ui-btn-xs" data-tpl-apply="${t.id}" data-mode="replace">Заменить</button>
              <button class="ui-btn ui-btn-light ui-btn-xs" data-tpl-rename="${t.id}">Переименовать</button>
              <button class="ui-btn ui-btn-danger ui-btn-xs" data-tpl-delete="${t.id}">Удалить</button>
            </div>
          </div>
        `;
      }).join('');

      return `
        <div id="${containerId}">
          <div class="secSearch">
            <input id="${containerId}_q" class="input" placeholder="Поиск шаблонов..." value="${BX.util.htmlspecialchars(q || '')}">
          </div>
          <div class="secGrid">${cards || '<div class="muted">Ничего не найдено</div>'}</div>
        </div>
      `;
    };

    BX.UI.Dialogs.MessageBox.show({
      title: 'Каталог секций',
      message: render(''),
      buttons: BX.UI.Dialogs.MessageBoxButtons.CANCEL,
      onCancel: function (mb) { mb.close(); }
    });

    setTimeout(() => {
      const root = document.getElementById(containerId);
      if (!root) {
        notify('Каталог секций: не найден контейнер');
        return;
      }

      const rerender = (q) => {
        root.outerHTML = render(q);
        const newRoot = document.getElementById(containerId);
        if (!newRoot) return;
        bind(newRoot);
      };

      const bind = (r) => {
        const q = document.getElementById(containerId + '_q');
        if (q) q.oninput = () => rerender(q.value);

        r.onclick = async (e) => {
          const applyBtn = e.target.closest('[data-tpl-apply]');
          if (applyBtn) {
            const tplId = parseInt(applyBtn.getAttribute('data-tpl-apply'), 10);
            const mode = applyBtn.getAttribute('data-mode') || 'append';
            try {
              const r2 = await api('template.applyToPage', { siteId, pageId, templateId: tplId, mode });
              if (!r2 || r2.ok !== true) { notify('Не удалось применить'); return; }
              notify('Готово: добавлено блоков ' + (r2.added || 0));
              loadBlocks();
            } catch (err) {
              notify('Ошибка apply');
            }
            return;
          }

          const renameBtn = e.target.closest('[data-tpl-rename]');
          if (renameBtn) {
            const tplId = parseInt(renameBtn.getAttribute('data-tpl-rename'), 10);
            const cur = templates.find(x => parseInt(x.id, 10) === tplId);

            BX.UI.Dialogs.MessageBox.show({
              title: 'Переименовать',
              message: `
                <div class="field">
                  <label>Название</label>
                  <input id="tpl_new_name" class="input" value="${BX.util.htmlspecialchars(cur?.name || '')}">
                </div>
              `,
              buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
              onOk: function (mb) {
                const name = (document.getElementById('tpl_new_name')?.value || '').trim();
                if (!name) { notify('Введите название'); return; }

                api('template.rename', { id: tplId, name })
                  .then(async x => {
                    if (!x || x.ok !== true) { notify('Не удалось переименовать'); return; }

                    notify('Переименовано');
                    mb.close();

                    const refreshed = await api('template.list', {});
                    templates = refreshed?.templates || templates;
                    rerender(q?.value || '');
                  })
                  .catch(() => notify('Ошибка template.rename'));
              }
            });
            return;
          }

          const delBtn = e.target.closest('[data-tpl-delete]');
          if (delBtn) {
            const tplId = parseInt(delBtn.getAttribute('data-tpl-delete'), 10);
            if (!confirm('Удалить шаблон #' + tplId + '?')) return;

            try {
              const x = await api('template.delete', { id: tplId });
              if (!x || x.ok !== true) { notify('Не удалось удалить'); return; }
              notify('Удалено');

              const refreshed = await api('template.list', {});
              templates = refreshed?.templates || templates;
              rerender(q?.value || '');
            } catch (err) {
              notify('Ошибка template.delete');
            }
            return;
          }
        };
      };

      bind(root);
    }, 0);
  }

  function addTextBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Text блок',
      message: `
        <div class="field">
          <label>Текст</label>
          <textarea id="new_text_value" class="input" style="height:180px;"></textarea>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb){
        const text = document.getElementById('new_text_value')?.value || '';
        api('block.create', { pageId, type:'text', text })
          .then(res => {
            if(!res || res.ok!==true){ notify('Не удалось создать text'); return; }
            notify('Text создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create (text)'));
      }
    });
  }

Пришлю третью часть следующим сообщением.