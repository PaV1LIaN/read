Идём дальше.

Создай файл:

/local/sitebuilder/assets/js/editor.sections.js

и вставь туда это:

window.SBEditor = window.SBEditor || {};

window.SBEditor.SECTION_PRESETS = {
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

window.SBEditor.sectionPresetOptions = function (selected = 'default') {
  return `
    <option value="default" ${selected === 'default' ? 'selected' : ''}>Default</option>
    <option value="hero" ${selected === 'hero' ? 'selected' : ''}>Hero</option>
    <option value="light" ${selected === 'light' ? 'selected' : ''}>Light</option>
    <option value="accent" ${selected === 'accent' ? 'selected' : ''}>Accent</option>
    <option value="card" ${selected === 'card' ? 'selected' : ''}>Card</option>
  `;
};

window.SBEditor.applySectionPresetToForm = function (presetKey, suffix = '') {
  const preset = window.SBEditor.SECTION_PRESETS[presetKey] || window.SBEditor.SECTION_PRESETS.default;

  const boxedEl = document.getElementById('sec_boxed' + suffix);
  const bgEl = document.getElementById('sec_bg' + suffix);
  const ptEl = document.getElementById('sec_pt' + suffix);
  const pbEl = document.getElementById('sec_pb' + suffix);
  const borderEl = document.getElementById('sec_border' + suffix);
  const radiusEl = document.getElementById('sec_radius' + suffix);

  if (boxedEl) boxedEl.checked = !!preset.boxed;
  if (bgEl) bgEl.value = preset.background;
  if (ptEl) ptEl.value = preset.paddingTop;
  if (pbEl) pbEl.value = preset.paddingBottom;
  if (borderEl) borderEl.checked = !!preset.border;
  if (radiusEl) radiusEl.value = preset.radius;
};

window.SBEditor.createBlockAfterSection = async function (sectionId, type, payload = {}) {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  const listRes = await api('block.list', { pageId: st.pageId });
  if (!listRes || listRes.ok !== true) {
    notify('Не удалось загрузить блоки страницы');
    return;
  }

  const blocks = Array.isArray(listRes.blocks) ? listRes.blocks.slice() : [];
  const sectionIndex = blocks.findIndex(b => parseInt(b.id, 10) === parseInt(sectionId, 10));

  if (sectionIndex < 0) {
    notify('Section не найдена');
    return;
  }

  const sectionSort = parseInt(blocks[sectionIndex].sort || 0, 10);

  let insertSort = sectionSort + 10;
  for (let i = sectionIndex + 1; i < blocks.length; i++) {
    const next = blocks[i];
    if ((next.type || '') === 'section') {
      insertSort = parseInt(next.sort || insertSort, 10) - 1;
      break;
    }
    insertSort = Math.max(insertSort, parseInt(next.sort || 0, 10) + 10);
  }

  const createRes = await api('block.create', Object.assign({
    pageId: st.pageId,
    type
  }, payload));

  if (!createRes || createRes.ok !== true || !createRes.block) {
    notify('Не удалось создать блок');
    return;
  }

  const newBlockId = parseInt(createRes.block.id, 10);

  const refreshRes = await api('block.list', { pageId: st.pageId });
  if (!refreshRes || refreshRes.ok !== true) {
    notify('Блок создан, но не удалось обновить порядок');
    if (typeof window.SBEditor.loadBlocks === 'function') {
      window.SBEditor.loadBlocks();
    }
    return;
  }

  const freshBlocks = Array.isArray(refreshRes.blocks) ? refreshRes.blocks.slice() : [];
  const moved = freshBlocks.find(b => parseInt(b.id, 10) === newBlockId);
  if (!moved) {
    if (typeof window.SBEditor.loadBlocks === 'function') {
      window.SBEditor.loadBlocks();
    }
    return;
  }

  const desiredOrder = freshBlocks
    .sort((a, b) => parseInt(a.sort || 0, 10) - parseInt(b.sort || 0, 10))
    .filter(b => parseInt(b.id, 10) !== newBlockId);

  let targetIndex = desiredOrder.findIndex(b => parseInt(b.sort || 0, 10) > insertSort);
  if (targetIndex < 0) targetIndex = desiredOrder.length;

  desiredOrder.splice(targetIndex, 0, moved);

  const order = desiredOrder.map(b => parseInt(b.id, 10));

  const reorderRes = await api('block.reorder', {
    pageId: st.pageId,
    order: JSON.stringify(order)
  });

  if (!reorderRes || reorderRes.ok !== true) {
    notify('Блок создан, но не удалось поставить после section');
    if (typeof window.SBEditor.loadBlocks === 'function') {
      window.SBEditor.loadBlocks();
    }
    return;
  }

  notify('Блок добавлен');
  if (typeof window.SBEditor.loadBlocks === 'function') {
    window.SBEditor.loadBlocks();
  }
};

window.SBEditor.quickAddHeadingAfterSection = function (sectionId) {
  return window.SBEditor.createBlockAfterSection(sectionId, 'heading', {
    text: 'Новый заголовок',
    level: 'h2',
    align: 'left'
  });
};

window.SBEditor.quickAddTextAfterSection = function (sectionId) {
  return window.SBEditor.createBlockAfterSection(sectionId, 'text', {
    text: 'Новый текст'
  });
};

window.SBEditor.quickAddButtonAfterSection = function (sectionId) {
  return window.SBEditor.createBlockAfterSection(sectionId, 'button', {
    text: 'Кнопка',
    url: '/',
    variant: 'primary'
  });
};

window.SBEditor.quickAddCardsAfterSection = function (sectionId) {
  return window.SBEditor.createBlockAfterSection(sectionId, 'cards', {
    columns: 3,
    items: JSON.stringify([
      { title: 'Карточка 1', text: 'Описание 1', imageFileId: 0, buttonText: '', buttonUrl: '' },
      { title: 'Карточка 2', text: 'Описание 2', imageFileId: 0, buttonText: '', buttonUrl: '' },
      { title: 'Карточка 3', text: 'Описание 3', imageFileId: 0, buttonText: '', buttonUrl: '' }
    ])
  });
};

window.SBEditor.addSectionBlock = function () {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  BX.UI.Dialogs.MessageBox.show({
    title: 'Новая Section',
    message: `
      <div>
        <div class="field">
          <label>Пресет</label>
          <select id="sec_preset" class="input">
            ${window.SBEditor.sectionPresetOptions('default')}
          </select>
        </div>

        <div class="field">
          <label><input id="sec_boxed" type="checkbox" checked> Ограничить по контейнеру</label>
        </div>

        <div class="field">
          <label>Цвет фона</label>
          <input id="sec_bg" class="input" value="#FFFFFF" placeholder="#FFFFFF" />
        </div>

        <div class="field">
          <label>Отступ сверху (0..200)</label>
          <input id="sec_pt" class="input" type="number" min="0" max="200" value="32" />
        </div>

        <div class="field">
          <label>Отступ снизу (0..200)</label>
          <input id="sec_pb" class="input" type="number" min="0" max="200" value="32" />
        </div>

        <div class="field">
          <label><input id="sec_border" type="checkbox"> Показать рамку</label>
        </div>

        <div class="field">
          <label>Скругление (0..40)</label>
          <input id="sec_radius" class="input" type="number" min="0" max="40" value="0" />
        </div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function (mbox) {
      const boxed = document.getElementById('sec_boxed')?.checked ? '1' : '0';
      const background = (document.getElementById('sec_bg')?.value || '#FFFFFF').trim();
      const paddingTop = parseInt(document.getElementById('sec_pt')?.value || '32', 10);
      const paddingBottom = parseInt(document.getElementById('sec_pb')?.value || '32', 10);
      const border = document.getElementById('sec_border')?.checked ? '1' : '0';
      const radius = parseInt(document.getElementById('sec_radius')?.value || '0', 10);

      api('block.create', {
        pageId: st.pageId,
        type: 'section',
        boxed,
        background,
        paddingTop,
        paddingBottom,
        border,
        radius
      })
        .then(res => {
          if (!res || res.ok !== true) {
            notify('Не удалось создать section');
            return;
          }
          notify('Section создана');
          mbox.close();
          if (typeof window.SBEditor.loadBlocks === 'function') {
            window.SBEditor.loadBlocks();
          }
        })
        .catch(() => notify('Ошибка block.create (section)'));
    }
  });

  setTimeout(() => {
    const presetEl = document.getElementById('sec_preset');
    if (presetEl) {
      presetEl.addEventListener('change', () => {
        window.SBEditor.applySectionPresetToForm(presetEl.value);
      });
      window.SBEditor.applySectionPresetToForm('default');
    }
  }, 0);
};

window.SBEditor.editSectionBlock = function (id) {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  api('block.list', { pageId: st.pageId }).then(res => {
    if (!res || res.ok !== true) return;

    const blk = (res.blocks || []).find(x => parseInt(x.id, 10) === id);
    const cur = (blk && blk.content && typeof blk.content === 'object') ? blk.content : {};

    BX.UI.Dialogs.MessageBox.show({
      title: 'Редактировать Section #' + id,
      message: `
        <div>
          <div class="field">
            <label>Пресет</label>
            <select id="sec_preset_e" class="input">
              ${window.SBEditor.sectionPresetOptions('default')}
            </select>
          </div>

          <div class="field">
            <label><input id="sec_boxed_e" type="checkbox" ${cur.boxed ? 'checked' : ''}> Ограничить по контейнеру</label>
          </div>

          <div class="field">
            <label>Цвет фона</label>
            <input id="sec_bg_e" class="input" value="${BX.util.htmlspecialchars(cur.background || '#FFFFFF')}" />
          </div>

          <div class="field">
            <label>Отступ сверху (0..200)</label>
            <input id="sec_pt_e" class="input" type="number" min="0" max="200" value="${parseInt(cur.paddingTop || 32, 10)}" />
          </div>

          <div class="field">
            <label>Отступ снизу (0..200)</label>
            <input id="sec_pb_e" class="input" type="number" min="0" max="200" value="${parseInt(cur.paddingBottom || 32, 10)}" />
          </div>

          <div class="field">
            <label><input id="sec_border_e" type="checkbox" ${cur.border ? 'checked' : ''}> Показать рамку</label>
          </div>

          <div class="field">
            <label>Скругление (0..40)</label>
            <input id="sec_radius_e" class="input" type="number" min="0" max="40" value="${parseInt(cur.radius || 0, 10)}" />
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mbox) {
        const boxed = document.getElementById('sec_boxed_e')?.checked ? '1' : '0';
        const background = (document.getElementById('sec_bg_e')?.value || '#FFFFFF').trim();
        const paddingTop = parseInt(document.getElementById('sec_pt_e')?.value || '32', 10);
        const paddingBottom = parseInt(document.getElementById('sec_pb_e')?.value || '32', 10);
        const border = document.getElementById('sec_border_e')?.checked ? '1' : '0';
        const radius = parseInt(document.getElementById('sec_radius_e')?.value || '0', 10);

        api('block.update', {
          id,
          boxed,
          background,
          paddingTop,
          paddingBottom,
          border,
          radius
        })
          .then(r => {
            if (!r || r.ok !== true) {
              notify('Не удалось сохранить section');
              return;
            }
            notify('Section сохранена');
            mbox.close();
            if (typeof window.SBEditor.loadBlocks === 'function') {
              window.SBEditor.loadBlocks();
            }
          })
          .catch(() => notify('Ошибка block.update (section)'));
      }
    });

    setTimeout(() => {
      const presetEl = document.getElementById('sec_preset_e');
      if (presetEl) {
        presetEl.addEventListener('change', () => {
          window.SBEditor.applySectionPresetToForm(presetEl.value, '_e');
        });
      }
    }, 0);
  });
};

Дальше нужен editor.dnd.js. Создай:

/local/sitebuilder/assets/js/editor.dnd.js

и я дам его следующим сообщением.