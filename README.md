Пропустил, да. Вот часть 1 из 8:

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

Дальше уже у тебя есть часть 2. Если хочешь, я могу сразу продолжить частью 3 в следующем сообщении.