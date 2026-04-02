Продолжение третьей части с того места, где оборвалось:

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

Шлю следующую часть дальше.