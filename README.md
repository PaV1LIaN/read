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

  const u = document.getElementById('btn_url');
      const v = document.getElementById('btn_variant');
      const p = document.getElementById('btn_preview');
      if (!t || !u || !v || !p) return;

      const update = () => {
        p.textContent = t.value || 'Кнопка';
        p.setAttribute('href', u.value || '#');
        p.className = 'btnPreview ' + (v.value === 'secondary' ? 'btnSecondary' : 'btnPrimary');
      };

      t.addEventListener('input', update);
      u.addEventListener('input', update);
      v.addEventListener('change', update);
      update();
    }, 0);
  }

  function addHeadingBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Heading блок',
      message: `
        <div>
          <div class="field">
            <label>Текст</label>
            <input id="h_text" class="input" placeholder="Заголовок" />
          </div>
          <div class="grid2">
            <div class="field">
              <label>Уровень</label>
              <select id="h_level" class="input">
                <option value="h1">h1</option>
                <option value="h2" selected>h2</option>
                <option value="h3">h3</option>
              </select>
            </div>
            <div class="field">
              <label>Выравнивание</label>
              <select id="h_align" class="input">
                <option value="left" selected>left</option>
                <option value="center">center</option>
                <option value="right">right</option>
              </select>
            </div>
          </div>
          <div class="muted" style="margin-top:10px;">Превью:</div>
          <div id="h_preview_wrap" style="border:1px solid #e5e7eb;border-radius:10px;padding:12px;background:#fff;">
            <h2 id="h_preview">Заголовок</h2>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = (document.getElementById('h_text')?.value || '').trim();
        const level = (document.getElementById('h_level')?.value || 'h2');
        const align = (document.getElementById('h_align')?.value || 'left');

        if (!text) { notify('Введите текст'); return; }

        api('block.create', { pageId, type: 'heading', text, level, align })
          .then(res => {
            if (!res || res.ok !== true) { notify('Не удалось создать heading-блок'); return; }
            notify('Heading-блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create (heading)'));
      }
    });

    setTimeout(() => {
      const t = document.getElementById('h_text');
      const l = document.getElementById('h_level');
      const a = document.getElementById('h_align');
      const wrap = document.getElementById('h_preview_wrap');
      if (!t || !l || !a || !wrap) return;

      const update = () => {
        const txt = t.value || 'Заголовок';
        const tag = headingTag(l.value);
        const al = headingAlign(a.value);
        wrap.style.textAlign = al;
        wrap.innerHTML = `<${tag} id="h_preview">${BX.util.htmlspecialchars(txt)}</${tag}>`;
      };

      t.addEventListener('input', update);
      l.addEventListener('change', update);
      a.addEventListener('change', update);
      update();
    }, 0);
  }

  function addCols2Block() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Columns2 блок',
      message: `
        <div>
          <div class="field">
            <label>Левая колонка</label>
            <textarea id="c_left" class="input" style="height:120px;"></textarea>
          </div>
          <div class="field">
            <label>Правая колонка</label>
            <textarea id="c_right" class="input" style="height:120px;"></textarea>
          </div>
          <div class="field">
            <label>Соотношение</label>
            <select id="c_ratio" class="input">
              <option value="50-50" selected>50 / 50</option>
              <option value="33-67">33 / 67</option>
              <option value="67-33">67 / 33</option>
            </select>
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const left  = document.getElementById('c_left')?.value || '';
        const right = document.getElementById('c_right')?.value || '';
        const ratio = document.getElementById('c_ratio')?.value || '50-50';

        api('block.create', { pageId, type:'columns2', left, right, ratio })
          .then(res => {
            if (!res || res.ok !== true) { notify('Не удалось создать columns2'); return; }
            notify('Columns2 создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create (columns2)'));
      }
    });
  }

  async function openGalleryDialog(mode, blockId, currentContent) {
    const currentCols = currentContent?.columns ? parseInt(currentContent.columns, 10) : 3;
    const currentImages = Array.isArray(currentContent?.images) ? currentContent.images : [];

    BX.UI.Dialogs.MessageBox.show({
      title: mode === 'edit' ? ('Редактировать Gallery #' + blockId) : 'Новый Gallery блок',
      message: `
        <div>
          <div class="field">
            <label>Колонки</label>
            <select id="g_cols" class="input">
              <option value="2" ${currentCols===2?'selected':''}>2</option>
              <option value="3" ${currentCols===3?'selected':''}>3</option>
              <option value="4" ${currentCols===4?'selected':''}>4</option>
            </select>
          </div>

          <div class="muted" style="margin-top:8px;">Выбери файлы из “Файлы” сайта:</div>
          <div id="g_list" class="galPick">Загрузка списка...</div>

          <div class="muted" style="margin-top:10px;">Превью:</div>
          <div id="g_prev" class="galPrev"></div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb){
        const cols = parseInt(document.getElementById('g_cols')?.value || '3', 10);
        const list = document.getElementById('g_list');
        if (!list) return;

        const checks = list.querySelectorAll('input[type="checkbox"][data-fid]');
        const selected = [];
        checks.forEach(ch => {
          if (!ch.checked) return;
          const fid = parseInt(ch.getAttribute('data-fid'), 10);
          const altEl = list.querySelector(`input[data-alt-for="${fid}"]`);
          const alt = (altEl?.value || '').trim();
          if (fid) selected.push({ fileId: fid, alt });
        });

        if (!selected.length) { notify('Выбери хотя бы 1 файл'); return; }

        const images = JSON.stringify(selected);
        const payload = { columns: cols, images };

        const call = (mode === 'edit')
          ? api('block.update', Object.assign({ id: blockId }, payload))
          : api('block.create', Object.assign({ pageId, type:'gallery' }, payload));

        call.then(res => {
          if (!res || res.ok !== true) { notify('Не удалось сохранить gallery'); return; }
          notify(mode==='edit' ? 'Сохранено' : 'Gallery создан');
          mb.close();
          loadBlocks();
        }).catch(() => notify('Ошибка запроса gallery'));
      }
    });

    setTimeout(async () => {
      const box = document.getElementById('g_list');
      const prev = document.getElementById('g_prev');
      const colsSel = document.getElementById('g_cols');
      if (!box || !prev || !colsSel) return;

      const selectedMap = {};
      currentImages.forEach(it => { selectedMap[parseInt(it.fileId,10)] = (it.alt || ''); });

      try {
        const files = await getFilesForSite();
        if (!files.length) { box.innerHTML = '<div class="muted">Файлов нет (загрузите в “Файлы”)</div>'; return; }

        box.innerHTML = files.map(f => {
          const checked = selectedMap[f.id] !== undefined ? 'checked' : '';
          const altVal = selectedMap[f.id] !== undefined ? selectedMap[f.id] : '';
          return `
            <div class="row">
              <input type="checkbox" data-fid="${f.id}" ${checked}>
              <div style="flex:1;">
                <div><b>${BX.util.htmlspecialchars(f.name)}</b> <small>(id ${f.id})</small></div>
                <input class="input" style="margin-top:6px;" data-alt-for="${f.id}" placeholder="alt (опционально)" value="${BX.util.htmlspecialchars(altVal)}">
              </div>
            </div>
          `;
        }).join('');

        const renderPrev = () => {
          const cols = parseInt(colsSel.value || '3', 10);
          prev.style.gridTemplateColumns = galleryTemplate(cols);

          const checks = box.querySelectorAll('input[type="checkbox"][data-fid]');
          let html = '';
          checks.forEach(ch => {
            if (!ch.checked) return;
            const fid = parseInt(ch.getAttribute('data-fid'), 10);
            if (!fid) return;
            html += `<img src="${fileDownloadUrl(fid)}" alt="">`;
          });
          prev.innerHTML = html || '<div class="muted">Ничего не выбрано</div>';
        };

        box.addEventListener('change', renderPrev);
        colsSel.addEventListener('change', renderPrev);
        renderPrev();
      } catch (e) {
        box.innerHTML = '<div class="muted">Ошибка загрузки файлов</div>';
      }
    }, 0);
  }

  function addGalleryBlock() { openGalleryDialog('create', 0, null); }

  async function openCardDialog(mode, blockId, current) {
    const curTitle = current?.title || '';
    const curText = current?.text || '';
    const curImage = current?.imageFileId ? parseInt(current.imageFileId, 10) : 0;
    const curBtnText = current?.buttonText || '';
    const curBtnUrl = current?.buttonUrl || '';

    BX.UI.Dialogs.MessageBox.show({
      title: mode === 'edit' ? ('Редактировать Card #' + blockId) : 'Новый Card блок',
      message: `
        <div>
          <div class="field">
            <label>Заголовок</label>
            <input id="c_title" class="input" value="${BX.util.htmlspecialchars(curTitle)}">
          </div>
          <div class="field">
            <label>Текст</label>
            <textarea id="c_text" class="input" style="height:120px;">${BX.util.htmlspecialchars(curText)}</textarea>
          </div>

          <div class="field">
            <label>Картинка (из файлов сайта, опционально)</label>
            <select id="c_img" class="input"><option value="">Загрузка списка...</option></select>
          </div>
          <div id="c_img_prev" class="imgPrev" style="display:${curImage? 'block':'none'};">
            <img id="c_img_prev_img" src="${curImage ? fileDownloadUrl(curImage) : ''}" alt="">
          </div>

          <div class="field">
            <label>Текст кнопки (опционально)</label>
            <input id="c_btn_text" class="input" value="${BX.util.htmlspecialchars(curBtnText)}" placeholder="например: Подробнее">
          </div>
          <div class="field">
            <label>URL кнопки (опционально)</label>
            <input id="c_btn_url" class="input" value="${BX.util.htmlspecialchars(curBtnUrl)}" placeholder="https://... или /local/...">
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function(mb) {
        const title = (document.getElementById('c_title')?.value || '').trim();
        const text = (document.getElementById('c_text')?.value || '');
        const imageFileId = parseInt(document.getElementById('c_img')?.value || '0', 10);
        const buttonText = (document.getElementById('c_btn_text')?.value || '').trim();
        const buttonUrl = (document.getElementById('c_btn_url')?.value || '').trim();

        if (!title) { notify('Введите заголовок'); return; }

        const payload = { title, text, imageFileId, buttonText, buttonUrl };
        const call = (mode === 'edit')
          ? api('block.update', Object.assign({ id: blockId }, payload))
          : api('block.create', Object.assign({ pageId, type:'card' }, payload));

        call.then(res => {
          if (!res || res.ok !== true) { notify('Не удалось сохранить card'); return; }
          notify(mode === 'edit' ? 'Сохранено' : 'Card создан');
          mb.close();
          loadBlocks();
        }).catch(() => notify('Ошибка запроса card'));
      }
    });

    setTimeout(async () => {
      const sel = document.getElementById('c_img');
      const prevWrap = document.getElementById('c_img_prev');
      const prevImg = document.getElementById('c_img_prev_img');
      if (!sel || !prevWrap || !prevImg) return;

      try {
        const files = await getFilesForSite();
        sel.innerHTML = '<option value="0">— без картинки —</option>' + files.map(f => {
          const s = (parseInt(f.id,10) === curImage) ? 'selected' : '';
          return `<option value="${f.id}" ${s}>${BX.util.htmlspecialchars(f.name)} (id ${f.id})</option>`;
        }).join('');

        const updatePrev = () => {
          const fid = parseInt(sel.value || '0', 10);
          if (!fid) { prevWrap.style.display = 'none'; prevImg.src = ''; return; }
          prevWrap.style.display = 'block';
          prevImg.src = fileDownloadUrl(fid);
        };
        sel.addEventListener('change', updatePrev);
        updatePrev();
      } catch (e) {
        sel.innerHTML = '<option value="0">Ошибка загрузки файлов</option>';
      }
    }, 0);
  }

  function addCardBlock() { openCardDialog('create', 0, null); }

  function editTextBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
      const current = blk && blk.content ? (blk.content.text || '') : '';

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Text #' + id,
        message: `<textarea id="edit_text" style="width:100%;height:160px;padding:8px;border:1px solid #d0d7de;border-radius:8px;">${BX.util.htmlspecialchars(current)}</textarea>`,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          const text = document.getElementById('edit_text')?.value ?? '';
          api('block.update', { id, text })
            .then(r => {
              if (!r || r.ok !== true) { notify('Не удалось сохранить'); return; }
              notify('Сохранено');
              mb.close();
              loadBlocks();
            })
            .catch(() => notify('Ошибка block.update'));
        }
      });
    });
  }

  function editImageBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
      const curFileId = blk && blk.content ? parseInt(blk.content.fileId || 0, 10) : 0;
      const curAlt = blk && blk.content ? (blk.content.alt || '') : '';

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Image #' + id,
        message: `
          <div>
            <div class="field">
              <label>Файл</label>
              <select id="edit_img_file" class="input">
                <option value="">Загрузка списка...</option>
              </select>
            </div>
            <div class="field">
              <label>ALT</label>
              <input id="edit_img_alt" class="input" value="${BX.util.htmlspecialchars(curAlt)}" />
            </div>
            <div id="edit_img_preview" class="imgPrev" style="display:${curFileId ? 'block':'none'};">
              <img id="edit_img_preview_img" src="${curFileId ? fileDownloadUrl(curFileId) : ''}" alt="">
            </div>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          const fileId = parseInt(document.getElementById('edit_img_file')?.value || '0', 10);
          const alt = (document.getElementById('edit_img_alt')?.value || '').trim();
          if (!fileId) { notify('Выбери файл'); return; }

          api('block.update', { id, fileId, alt })
            .then(r => {
              if (!r || r.ok !== true) { notify('Не удалось сохранить image-блок'); return; }
              notify('Сохранено');
              mb.close();
              loadBlocks();
            })
            .catch(() => notify('Ошибка block.update (image)'));
        }
      });

      setTimeout(async function () {
        const sel = document.getElementById('edit_img_file');
        if (!sel) return;

        try {
          const files = await getFilesForSite();
          if (!files.length) { sel.innerHTML = '<option value="">Файлов нет</option>'; return; }

          sel.innerHTML = '<option value="">— Выберите файл —</option>' + files.map(f => {
            const selected = (parseInt(f.id,10) === curFileId) ? 'selected' : '';
            return `<option value="${f.id}" ${selected}>${BX.util.htmlspecialchars(f.name)} (${f.id})</option>`;
          }).join('');

          sel.addEventListener('change', function () {
            const fid = parseInt(sel.value || '0', 10);
            const wrap = document.getElementById('edit_img_preview');
            const img = document.getElementById('edit_img_preview_img');
            if (!wrap || !img) return;
            if (!fid) { wrap.style.display = 'none'; img.src = ''; return; }
            wrap.style.display = 'block';
            img.src = fileDownloadUrl(fid);
          });
        } catch (e) {
          sel.innerHTML = '<option value="">Ошибка загрузки файлов</option>';
          notify('Не удалось получить список файлов');
        }
      }, 0);
    });
  }

  function editButtonBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

      const curText = blk && blk.content ? (blk.content.text || '') : '';
      const curUrl = blk && blk.content ? (blk.content.url || '') : '';
      const curVariant = blk && blk.content ? (blk.content.variant || 'primary') : 'primary';

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Button #' + id,
        message: `
          <div>
            <div class="field">
              <label>Текст</label>
              <input id="edit_btn_text" class="input" value="${BX.util.htmlspecialchars(curText)}" />
            </div>
            <div class="field">
              <label>URL</label>
              <input id="edit_btn_url" class="input" value="${BX.util.htmlspecialchars(curUrl)}" />
            </div>
            <div class="field">
              <label>Вариант</label>
              <select id="edit_btn_variant" class="input">
                <option value="primary" ${curVariant === 'primary' ? 'selected' : ''}>primary</option>
                <option value="secondary" ${curVariant === 'secondary' ? 'selected' : ''}>secondary</option>
              </select>
            </div>
            <div class="muted" style="margin-top:10px;">Превью:</div>
            <a id="edit_btn_preview" class="${btnClass(curVariant)}" href="${BX.util.htmlspecialchars(curUrl)}" target="_blank" rel="noopener noreferrer">
              ${BX.util.htmlspecialchars(curText || 'Кнопка')}
            </a>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          const text = (document.getElementById('edit_btn_text')?.value || '').trim();
          const url  = (document.getElementById('edit_btn_url')?.value || '').trim();
          const variant = (document.getElementById('edit_btn_variant')?.value || 'primary');

          if (!text) { notify('Введите текст'); return; }
          if (!url)  { notify('Введите URL'); return; }

          api('block.update', { id, text, url, variant })
            .then(r => {
              if (!r || r.ok !== true) { notify('Не удалось сохранить button-блок'); return; }
              notify('Сохранено');
              mb.close();
              loadBlocks();
            })
            .catch(() => notify('Ошибка block.update (button)'));
        }
      });

      setTimeout(() => {
        const t = document.getElementById('edit_btn_text');
        const u = document.getElementById('edit_btn_url');
        const v = document.getElementById('edit_btn_variant');
        const p = document.getElementById('edit_btn_preview');
        if (!t || !u || !v || !p) return;

        const update = () => {
          p.textContent = t.value || 'Кнопка';
          p.href = u.value || '#';
          p.className = btnClass(v.value);
        };

        t.addEventListener('input', update);
        u.addEventListener('input', update);
        v.addEventListener('change', update);
        update();
      }, 0);
    });
  }

  function editHeadingBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

      const curText = blk && blk.content ? (blk.content.text || '') : '';
      const curLevel = blk && blk.content ? (blk.content.level || 'h2') : 'h2';
      const curAlign = blk && blk.content ? (blk.content.align || 'left') : 'left';

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Heading #' + id,
        message: `
          <div>
            <div class="field">
              <label>Текст</label>
              <input id="edit_h_text" class="input" value="${BX.util.htmlspecialchars(curText)}" />
            </div>
            <div class="field">
              <label>Уровень</label>
              <select id="edit_h_level" class="input">
                <option value="h1" ${curLevel === 'h1' ? 'selected' : ''}>h1</option>
                <option value="h2" ${curLevel === 'h2' ? 'selected' : ''}>h2</option>
                <option value="h3" ${curLevel === 'h3' ? 'selected' : ''}>h3</option>
              </select>
            </div>
            <div class="field">
              <label>Выравнивание</label>
              <select id="edit_h_align" class="input">
                <option value="left" ${curAlign === 'left' ? 'selected' : ''}>left</option>
                <option value="center" ${curAlign === 'center' ? 'selected' : ''}>center</option>
                <option value="right" ${curAlign === 'right' ? 'selected' : ''}>right</option>
              </select>
            </div>
            <div class="muted" style="margin-top:10px;">Превью:</div>
            <div id="edit_h_preview_wrap" class="headingPreview" style="text-align:${BX.util.htmlspecialchars(curAlign)};">
              <${headingTag(curLevel)} id="edit_h_preview">${BX.util.htmlspecialchars(curText || 'Заголовок')}</${headingTag(curLevel)}>
            </div>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          const text = (document.getElementById('edit_h_text')?.value || '').trim();
          const level = (document.getElementById('edit_h_level')?.value || 'h2');
          const align = (document.getElementById('edit_h_align')?.value || 'left');

          if (!text) { notify('Введите текст'); return; }

          api('block.update', { id, text, level, align })
            .then(r => {
              if (!r || r.ok !== true) { notify('Не удалось сохранить heading'); return; }
              notify('Сохранено');
              mb.close();
              loadBlocks();
            })
            .catch(() => notify('Ошибка block.update (heading)'));
        }
      });

      setTimeout(() => {
        const t = document.getElementById('edit_h_text');
        const l = document.getElementById('edit_h_level');
        const a = document.getElementById('edit_h_align');
        const wrap = document.getElementById('edit_h_preview_wrap');
        if (!t || !l || !a || !wrap) return;

        const update = () => {
          const txt = t.value || 'Заголовок';
          const tag = headingTag(l.value);
          const al = headingAlign(a.value);
          wrap.style.textAlign = al;
          wrap.innerHTML = `<${tag} id="edit_h_preview">${BX.util.htmlspecialchars(txt)}</${tag}>`;
        };

        t.addEventListener('input', update);
        l.addEventListener('change', update);
        a.addEventListener('change', update);
        update();
      }, 0);
    });
  }

  function editCols2Block(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

      const curRatio = blk && blk.content ? (blk.content.ratio || '50-50') : '50-50';
      const curLeft = blk && blk.content ? (blk.content.left || '') : '';
      const curRight = blk && blk.content ? (blk.content.right || '') : '';

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Columns2 #' + id,
        message: `
          <div>
            <div class="field">
              <label>Соотношение</label>
              <select id="ec_ratio" class="input">
                <option value="50-50" ${curRatio==='50-50'?'selected':''}>50 / 50</option>
                <option value="33-67" ${curRatio==='33-67'?'selected':''}>33 / 67</option>
                <option value="67-33" ${curRatio==='67-33'?'selected':''}>67 / 33</option>
              </select>
            </div>
            <div class="field">
              <label>Левая колонка (текст)</label>
              <textarea id="ec_left" class="input" style="height:120px;">${BX.util.htmlspecialchars(curLeft)}</textarea>
            </div>
            <div class="field">
              <label>Правая колонка (текст)</label>
              <textarea id="ec_right" class="input" style="height:120px;">${BX.util.htmlspecialchars(curRight)}</textarea>
            </div>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          const ratio = (document.getElementById('ec_ratio')?.value || '50-50');
          const left = (document.getElementById('ec_left')?.value || '');
          const right = (document.getElementById('ec_right')?.value || '');

          api('block.update', { id, ratio, left, right })
            .then(r => {
              if (!r || r.ok !== true) { notify('Не удалось сохранить columns2'); return; }
              notify('Сохранено');
              mb.close();
              loadBlocks();
            })
            .catch(() => notify('Ошибка block.update (columns2)'));
        }
      });
    });
  }

  function editSpacerBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
      const curH = blk && blk.content ? parseInt(blk.content.height || 40, 10) : 40;
      const curLine = blk && blk.content ? (blk.content.line === true || blk.content.line === 'true') : false;

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Spacer #' + id,
        message: `
          <div>
            <div class="field">
              <label>Высота (10..200 px)</label>
              <input id="esp_h" class="input" type="number" min="10" max="200" value="${curH}" />
            </div>
            <div class="field">
              <label><input id="esp_line" type="checkbox" ${curLine ? 'checked':''} /> Рисовать линию</label>
            </div>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function(mb){
          const height = parseInt(document.getElementById('esp_h')?.value || String(curH), 10);
          const line = document.getElementById('esp_line')?.checked ? '1' : '0';

          api('block.update', { id, height, line })
            .then(r => {
              if (!r || r.ok !== true) { notify('Не удалось сохранить spacer'); return; }
              notify('Сохранено');
              mb.close();
              loadBlocks();
            })
            .catch(() => notify('Ошибка block.update (spacer)'));
        }
      });
    });
  }

  function editGalleryBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
      openGalleryDialog('edit', id, blk?.content || null);
    });
  }

  function editCardBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
      openCardDialog('edit', id, blk?.content || null);
    });
  }

  function addCardsBlock() {
    openCardsBuilderDialog('create', 0, { columns: 3, items: [
      { title: 'Преимущество 1', text: 'Короткое описание' },
      { title: 'Преимущество 2', text: 'Короткое описание' },
      { title: 'Преимущество 3', text: 'Короткое описание' }
    ]});
  }

  function editCardsBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
      openCardsBuilderDialog('edit', id, blk?.content || null);
    }).catch(() => notify('Ошибка block.list'));
  }

  window.SBEditor.saveTemplateFromPage = saveTemplateFromPage;
  window.SBEditor.applyTemplateToPage = applyTemplateToPage;
  window.SBEditor.openSectionsLibrary = openSectionsLibrary;
  window.SBEditor.openCardsBuilderDialog = openCardsBuilderDialog;

  window.SBEditor.addTextBlock = addTextBlock;
  window.SBEditor.addImageBlock = addImageBlock;
  window.SBEditor.addButtonBlock = addButtonBlock;
  window.SBEditor.addHeadingBlock = addHeadingBlock;
  window.SBEditor.addCols2Block = addCols2Block;
  window.SBEditor.addGalleryBlock = addGalleryBlock;
  window.SBEditor.addSpacerBlock = addSpacerBlock;
  window.SBEditor.addCardBlock = addCardBlock;
  window.SBEditor.addCardsBlock = addCardsBlock;

  window.SBEditor.editTextBlock = editTextBlock;
  window.SBEditor.editImageBlock = editImageBlock;
  window.SBEditor.editButtonBlock = editButtonBlock;
  window.SBEditor.editHeadingBlock = editHeadingBlock;
  window.SBEditor.editCols2Block = editCols2Block;
  window.SBEditor.editGalleryBlock = editGalleryBlock;
  window.SBEditor.editSpacerBlock = editSpacerBlock;
  window.SBEditor.editCardBlock = editCardBlock;
  window.SBEditor.editCardsBlock = editCardsBlock;

  window.saveTemplateFromPage = saveTemplateFromPage;
  window.applyTemplateToPage = applyTemplateToPage;
  window.openSectionsLibrary = openSectionsLibrary;
  window.openCardsBuilderDialog = openCardsBuilderDialog;
  window.addTextBlock = addTextBlock;
  window.addImageBlock = addImageBlock;
  window.addButtonBlock = addButtonBlock;
  window.addHeadingBlock = addHeadingBlock;
  window.addCols2Block = addCols2Block;
  window.addGalleryBlock = addGalleryBlock;
  window.addSpacerBlock = addSpacerBlock;
  window.addCardBlock = addCardBlock;
  window.addCardsBlock = addCardsBlock;
  window.editTextBlock = editTextBlock;
  window.editImageBlock = editImageBlock;
  window.editButtonBlock = editButtonBlock;
  window.editHeadingBlock = editHeadingBlock;
  window.editCols2Block = editCols2Block;
  window.editGalleryBlock = editGalleryBlock;
  window.editSpacerBlock = editSpacerBlock;
  window.editCardBlock = editCardBlock;
  window.editCardsBlock = editCardsBlock;
})();
