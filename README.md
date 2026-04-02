Чтобы не путаться с частями, ниже даю цельный хвост файла от openGalleryDialog(...) до конца.
Этого достаточно, чтобы доклеить всё после твоего текущего куска.

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

Если хочешь, следующим сообщением я скажу, где у тебя сейчас шов между частями, чтобы ты просто склеил их без дублей.