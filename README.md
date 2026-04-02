Часть 7 из 8:

window.SBEditor = window.SBEditor || {};

window.SBEditor.editTextBlock = function (id) {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  api('block.list', { pageId: st.pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

    BX.UI.Dialogs.MessageBox.show({
      title: 'Редактировать Text #' + id,
      message: `
        <div class="field">
          <label>Текст</label>
          <textarea id="edit_text_value" class="input" rows="12">${BX.util.htmlspecialchars(blk?.content?.text || '')}</textarea>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = document.getElementById('edit_text_value')?.value || '';

        api('block.update', { id, text })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось сохранить text'); return; }
            notify('Сохранено');
            mb.close();
            if (typeof window.SBEditor.loadBlocks === 'function') {
              window.SBEditor.loadBlocks();
            }
          })
          .catch(() => notify('Ошибка block.update (text)'));
      }
    });
  });
};

window.SBEditor.editImageBlock = async function (id) {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  let files = [];
  try { files = await window.SBEditor.getFilesForSite(); }
  catch (e) { notify('Не удалось загрузить файлы сайта'); return; }

  const res = await api('block.list', { pageId: st.pageId });
  if (!res || res.ok !== true) return;
  const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

  const curFile = parseInt(blk?.content?.fileId || 0, 10);
  const curAlt = blk?.content?.alt || '';

  const opts = ['<option value="0">— выбрать файл —</option>']
    .concat(files.map(f => `<option value="${f.id}" ${parseInt(f.id,10)===curFile?'selected':''}>${BX.util.htmlspecialchars(f.name)} (#${f.id})</option>`))
    .join('');

  BX.UI.Dialogs.MessageBox.show({
    title: 'Редактировать Image #' + id,
    message: `
      <div class="field">
        <label>Файл</label>
        <select id="edit_image_file" class="input">${opts}</select>
      </div>
      <div class="field">
        <label>Alt</label>
        <input id="edit_image_alt" class="input" value="${BX.util.htmlspecialchars(curAlt)}" />
      </div>
      <div id="edit_image_preview_wrap" style="margin-top:10px;"></div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function (mb) {
      const fileId = parseInt(document.getElementById('edit_image_file')?.value || '0', 10);
      const alt = document.getElementById('edit_image_alt')?.value || '';

      if (!fileId) { notify('Выбери файл'); return; }

      api('block.update', { id, fileId, alt })
        .then(r => {
          if (!r || r.ok !== true) { notify('Не удалось сохранить image'); return; }
          notify('Сохранено');
          mb.close();
          if (typeof window.SBEditor.loadBlocks === 'function') {
            window.SBEditor.loadBlocks();
          }
        })
        .catch(() => notify('Ошибка block.update (image)'));
    }
  });

  setTimeout(() => {
    const sel = document.getElementById('edit_image_file');
    const wrap = document.getElementById('edit_image_preview_wrap');
    if (!sel || !wrap) return;

    const renderPrev = () => {
      const fid = parseInt(sel.value || '0', 10);
      wrap.innerHTML = fid ? `<div class="imgPrev"><img src="${window.SBEditor.fileDownloadUrl(st.siteId, fid)}" alt=""></div>` : '';
    };

    sel.addEventListener('change', renderPrev);
    renderPrev();
  }, 0);
};

window.SBEditor.editButtonBlock = function (id) {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  api('block.list', { pageId: st.pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

    BX.UI.Dialogs.MessageBox.show({
      title: 'Редактировать Button #' + id,
      message: `
        <div class="field">
          <label>Текст кнопки</label>
          <input id="edit_btn_text" class="input" value="${BX.util.htmlspecialchars(blk?.content?.text || '')}" />
        </div>
        <div class="field">
          <label>URL</label>
          <input id="edit_btn_url" class="input" value="${BX.util.htmlspecialchars(blk?.content?.url || '/')}" />
        </div>
        <div class="field">
          <label>Вариант</label>
          <select id="edit_btn_variant" class="input">
            <option value="primary" ${(blk?.content?.variant || 'primary')==='primary'?'selected':''}>primary</option>
            <option value="secondary" ${(blk?.content?.variant || '')==='secondary'?'selected':''}>secondary</option>
          </select>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = document.getElementById('edit_btn_text')?.value || '';
        const url = document.getElementById('edit_btn_url')?.value || '';
        const variant = document.getElementById('edit_btn_variant')?.value || 'primary';

        api('block.update', { id, text, url, variant })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось сохранить button'); return; }
            notify('Сохранено');
            mb.close();
            if (typeof window.SBEditor.loadBlocks === 'function') {
              window.SBEditor.loadBlocks();
            }
          })
          .catch(() => notify('Ошибка block.update (button)'));
      }
    });
  });
};

window.SBEditor.editHeadingBlock = function (id) {
  const st = window.SBEditor.getState();
  const api = window.SBEditor.api;
  const notify = window.SBEditor.notify;

  api('block.list', { pageId: st.pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);

    BX.UI.Dialogs.MessageBox.show({
      title: 'Редактировать Heading #' + id,
      message: `
        <div class="field">
          <label>Текст</label>
          <input id="edit_h_text" class="input" value="${BX.util.htmlspecialchars(blk?.content?.text || '')}" />
        </div>
        <div class="field">
          <label>Уровень</label>
          <select id="edit_h_level" class="input">
            <option value="h1" ${(blk?.content?.level || '')==='h1'?'selected':''}>h1</option>
            <option value="h2" ${(blk?.content?.level || 'h2')==='h2'?'selected':''}>h2</option>
            <option value="h3" ${(blk?.content?.level || '')==='h3'?'selected':''}>h3</option>
          </select>
        </div>
        <div class="field">
          <label>Выравнивание</label>
          <select id="edit_h_align" class="input">
            <option value="left" ${(blk?.content?.align || 'left')==='left'?'selected':''}>left</option>
            <option value="center" ${(blk?.content?.align || '')==='center'?'selected':''}>center</option>
            <option value="right" ${(blk?.content?.align || '')==='right'?'selected':''}>right</option>
          </select>
        </div>
        <div id="edit_h_preview_wrap" style="margin-top:10px;border:1px dashed #d1d5db;border-radius:10px;padding:12px;background:#fff;"></div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
        const text = document.getElementById('edit_h_text')?.value || '';
        const level = document.getElementById('edit_h_level')?.value || 'h2';
        const align = document.getElementById('edit_h_align')?.value || 'left';

        api('block.update', { id, text, level, align })
          .then(r => {
            if (!r || r.ok !== true) { notify('Не удалось сохранить heading'); return; }
            notify('Сохранено');
            mb.close();
            if (typeof window.SBEditor.loadBlocks === 'function') {
              window.SBEditor.loadBlocks();
            }
          })
          .catch(() => notify('Ошибка block.update (heading)'));
      }
    });

Пришли дальше, и я отправлю часть 8 из 8.