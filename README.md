Часть 6 из 8:

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

Пришлю часть 7 следующим сообщением.