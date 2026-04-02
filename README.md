Продолжаю с того места, где у тебя обрывается editHeadingBlock(id):

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
      const current = blk?.content || {};

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Columns2 #' + id,
        message: `
          <div>
            <div class="field">
              <label>Соотношение</label>
              <select id="edit_c_ratio" class="input">
                <option value="50-50" ${(current.ratio || '50-50') === '50-50' ? 'selected' : ''}>50 / 50</option>
                <option value="33-67" ${(current.ratio || '') === '33-67' ? 'selected' : ''}>33 / 67</option>
                <option value="67-33" ${(current.ratio || '') === '67-33' ? 'selected' : ''}>67 / 33</option>
              </select>
            </div>
            <div class="field">
              <label>Левая колонка</label>
              <textarea id="edit_c_left" class="input" style="height:120px;">${BX.util.htmlspecialchars(current.left || '')}</textarea>
            </div>
            <div class="field">
              <label>Правая колонка</label>
              <textarea id="edit_c_right" class="input" style="height:120px;">${BX.util.htmlspecialchars(current.right || '')}</textarea>
            </div>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function (mb) {
          const ratio = document.getElementById('edit_c_ratio')?.value || '50-50';
          const left = document.getElementById('edit_c_left')?.value || '';
          const right = document.getElementById('edit_c_right')?.value || '';

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
      const current = blk?.content || {};

      BX.UI.Dialogs.MessageBox.show({
        title: 'Редактировать Spacer #' + id,
        message: `
          <div>
            <div class="field">
              <label>Высота (10..200 px)</label>
              <input id="edit_sp_h" class="input" type="number" min="10" max="200" value="${parseInt(current.height || 40, 10)}" />
            </div>
            <div class="field">
              <label><input id="edit_sp_line" type="checkbox" ${current.line ? 'checked' : ''} /> Рисовать линию</label>
            </div>
          </div>
        `,
        buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
        onOk: function(mb){
          const height = parseInt(document.getElementById('edit_sp_h')?.value || '40', 10);
          const line = document.getElementById('edit_sp_line')?.checked ? '1' : '0';

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

  function editCardsBlock(id) {
    api('block.list', { pageId }).then(res => {
      if (!res || res.ok !== true) return;
      const blk = (res.blocks || []).find(x => parseInt(x.id,10) === id);
      const current = blk?.content || {};

      openCardsBuilderDialog({
        title: 'Редактировать Cards #' + id,
        columns: parseInt(current.columns || 3, 10),
        items: Array.isArray(current.items) ? current.items : [],
        onSubmit: async function ({ columns, items }) {
          const r = await api('block.update', {
            id,
            columns,
            items: JSON.stringify(items)
          });

          if (!r || r.ok !== true) {
            notify('Не удалось сохранить cards');
            return;
          }

          notify('Cards сохранён');
          loadBlocks();
        }
      });
    });
  }

  window.saveTemplateFromPage = window.SBEditor.saveTemplateFromPage;
  window.applyTemplateToPage = window.SBEditor.applyTemplateToPage;
  window.openSectionsLibrary = window.SBEditor.openSectionsLibrary;
  window.openCardsBuilderDialog = window.SBEditor.openCardsBuilderDialog;

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

Если хочешь, я могу теперь собрать тебе цельный editor.dialogs.js одним сообщением в canmore-стиле блоками или помочь сразу проверить, нет ли в твоём текущем куске синтаксических дыр после склейки.