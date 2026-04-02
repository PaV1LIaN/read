Часть 4 из 8:

const align = document.getElementById('new_heading_align')?.value || 'left';

        api('block.create', { pageId, type: 'heading', text, level, align })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать heading'); return; }
            notify('Heading блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });
  }

  function addCols2Block() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Columns2',
      message: `
        <div class="field">
          <label>Левая колонка</label>
          <textarea id="new_cols2_left" class="input" rows="6"></textarea>
        </div>
        <div class="field">
          <label>Правая колонка</label>
          <textarea id="new_cols2_right" class="input" rows="6"></textarea>
        </div>
        <div class="field">
          <label>Соотношение</label>
          <select id="new_cols2_ratio" class="input">
            <option value="50-50" selected>50-50</option>
            <option value="33-67">33-67</option>
            <option value="67-33">67-33</option>
          </select>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const left = document.getElementById('new_cols2_left')?.value || '';
        const right = document.getElementById('new_cols2_right')?.value || '';
        const ratio = document.getElementById('new_cols2_ratio')?.value || '50-50';

        api('block.create', { pageId, type: 'columns2', left, right, ratio })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать columns2'); return; }
            notify('Columns2 блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });
  }

  async function addGalleryBlock() {
    let files = [];
    try { files = await getFilesForSite(); }
    catch (e) { notify('Не удалось загрузить файлы сайта'); return; }

    const options = files.map(f =>
      `<option value="${f.id}">${BX.util.htmlspecialchars(f.name)} (#${f.id})</option>`
    ).join('');

    BX.UI.Dialogs.MessageBox.show({
      title: 'Новая Gallery',
      message: `
        <div class="field">
          <label>Колонки</label>
          <select id="new_gallery_cols" class="input">
            <option value="2">2</option>
            <option value="3" selected>3</option>
            <option value="4">4</option>
          </select>
        </div>

        <div id="new_gallery_rows">
          <div class="field">
            <label>Изображение 1</label>
            <select class="input galleryFileSelect">${options}</select>
            <input class="input galleryAltInput" placeholder="Alt" style="margin-top:8px;" />
          </div>
        </div>

        <div style="margin-top:12px;">
          <button type="button" class="ui-btn ui-btn-light" id="gallery_add_row">+ Добавить изображение</button>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const columns = parseInt(document.getElementById('new_gallery_cols')?.value || '3', 10);
        const rows = Array.from(document.querySelectorAll('#new_gallery_rows .field'));

        const images = rows.map(row => ({
          fileId: parseInt(row.querySelector('.galleryFileSelect')?.value || '0', 10),
          alt: row.querySelector('.galleryAltInput')?.value || ''
        })).filter(x => x.fileId > 0);

        if (!images.length) { notify('Нужно выбрать хотя бы одно изображение'); return; }

        api('block.create', {
          pageId,
          type: 'gallery',
          columns,
          images: JSON.stringify(images)
        })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать gallery'); return; }
            notify('Gallery блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });

    setTimeout(() => {
      const wrap = document.getElementById('new_gallery_rows');
      const addBtn = document.getElementById('gallery_add_row');
      if (!wrap || !addBtn) return;

      addBtn.onclick = () => {
        const div = document.createElement('div');
        div.className = 'field';
        div.innerHTML = `
          <label>Изображение</label>
          <select class="input galleryFileSelect">${options}</select>
          <input class="input galleryAltInput" placeholder="Alt" style="margin-top:8px;" />
        `;
        wrap.appendChild(div);
      };
    }, 0);
  }

  function addSpacerBlock() {
    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Spacer',
      message: `
        <div class="field">
          <label>Высота</label>
          <input id="new_spacer_height" class="input" type="number" min="10" max="200" value="40" />
        </div>
        <div class="field">
          <label><input id="new_spacer_line" type="checkbox"> Показать линию</label>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const height = parseInt(document.getElementById('new_spacer_height')?.value || '40', 10);
        const line = document.getElementById('new_spacer_line')?.checked ? '1' : '0';

        api('block.create', { pageId, type: 'spacer', height, line })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать spacer'); return; }
            notify('Spacer блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });
  }

  async function addCardBlock() {
    let files = [];
    try { files = await getFilesForSite(); }
    catch (e) { notify('Не удалось загрузить файлы сайта'); return; }

    const opts = ['<option value="0">— без изображения —</option>']
      .concat(files.map(f => `<option value="${f.id}">${BX.util.htmlspecialchars(f.name)} (#${f.id})</option>`))
      .join('');

    BX.UI.Dialogs.MessageBox.show({
      title: 'Новый Card',
      message: `
        <div class="field">
          <label>Заголовок</label>
          <input id="new_card_title" class="input" />
        </div>
        <div class="field">
          <label>Текст</label>
          <textarea id="new_card_text" class="input" rows="6"></textarea>
        </div>
        <div class="field">
          <label>Изображение</label>
          <select id="new_card_image" class="input">${opts}</select>
        </div>
        <div class="field">
          <label>Текст кнопки</label>
          <input id="new_card_btn_text" class="input" />
        </div>
        <div class="field">
          <label>URL кнопки</label>
          <input id="new_card_btn_url" class="input" placeholder="/" />
        </div>
        <div id="new_card_preview" style="margin-top:10px;"></div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const title = document.getElementById('new_card_title')?.value || '';
        const text = document.getElementById('new_card_text')?.value || '';
        const imageFileId = parseInt(document.getElementById('new_card_image')?.value || '0', 10);
        const buttonText = document.getElementById('new_card_btn_text')?.value || '';
        const buttonUrl = document.getElementById('new_card_btn_url')?.value || '';

        api('block.create', {
          pageId,
          type: 'card',
          title,
          text,
          imageFileId,
          buttonText,
          buttonUrl
        })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось создать card'); return; }
            notify('Card блок создан');
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.create'));
      }
    });

    setTimeout(() => {
      const sel = document.getElementById('new_card_image');
      const wrap = document.getElementById('new_card_preview');
      if (!sel || !wrap) return;

      const renderPrev = () => {
        const fid = parseInt(sel.value || '0', 10);
        wrap.innerHTML = fid ? `<div class="imgPrev"><img src="${fileDownloadUrl(fid)}" alt=""></div>` : '';
      };

      sel.addEventListener('change', renderPrev);
      renderPrev();
    }, 0);
  }

  function addCardsBlock() {
    openCardsBuilderDialog({
      title: 'Новый Cards',
      columns: 3,
      items: [],
      onSubmit: async function ({ columns, items }) {
        const r = await api('block.create', {
          pageId,
          type: 'cards',
          columns,
          items: JSON.stringify(items)
        });

Пришлю часть 5 следующим сообщением.