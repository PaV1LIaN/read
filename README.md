Да, теперь видно точнее: в editor.php у тебя уже есть панель добавления блоков, но там пока нет section, и в заголовке списка типов тоже нет Section. Это видно в кнопках + Text ... + Cards и в описании доступных блоков в верхней карточке редактора. 

Сразу важное замечание: в загруженном архиве api.php всё ещё не содержит section в списке допустимых типов block.create. То есть одной правки editor.php будет мало — кнопка появится, но сохранение не пройдёт, пока не обновишь api.php. Это видно по массиву типов, где есть только text,image,button,heading,columns2,gallery,spacer,card,cards.

Что сделать в editor.php

1. Добавить кнопку + Section

Найди вверху, где сейчас:

<button class="ui-btn ui-btn-primary" id="btnAddText">+ Text</button>
<button class="ui-btn ui-btn-primary" id="btnAddImage">+ Image</button>
<button class="ui-btn ui-btn-primary" id="btnAddButton">+ Button</button>
<button class="ui-btn ui-btn-primary" id="btnAddHeading">+ Heading</button>
<button class="ui-btn ui-btn-primary" id="btnAddCols2">+ Columns2</button>
<button class="ui-btn ui-btn-primary" id="btnAddGallery">+ Gallery</button>
<button class="ui-btn ui-btn-primary" id="btnAddSpacer">+ Spacer</button>
<button class="ui-btn ui-btn-primary" id="btnAddCard">+ Card</button>
<button class="ui-btn ui-btn-primary" id="btnAddCards">+ Cards</button>

И добавь рядом:

<button class="ui-btn ui-btn-primary" id="btnAddSection">+ Section</button>

2. Обновить подпись доступных блоков

Где сейчас текст:

Блоки: <b>Text</b>, <b>Image</b>, <b>Button</b>, <b>Heading</b>, <b>Columns2</b>, <b>Gallery</b>, <b>Spacer</b>, <b>Card</b>.

Сделай так:

Блоки: <b>Section</b>, <b>Text</b>, <b>Image</b>, <b>Button</b>, <b>Heading</b>, <b>Columns2</b>, <b>Gallery</b>, <b>Spacer</b>, <b>Card</b>, <b>Cards</b>.

3. Завести ссылку на новую кнопку в JS

Рядом с остальными константами:

const btnAddText = document.getElementById('btnAddText');
const btnAddImage = document.getElementById('btnAddImage');
...
const btnAddCards = document.getElementById('btnAddCards');

добавь:

const btnAddSection = document.getElementById('btnAddSection');

4. Добавить диалог создания section

Вставь в editor.php рядом с функциями addSpacerBlock(), addGalleryBlock() и т.п. вот эту функцию:

function addSectionBlock() {
  BX.UI.Dialogs.MessageBox.show({
    title: 'Новая Section',
    message: `
      <div>
        <div class="field">
          <label><input id="sec_boxed" type="checkbox" checked> Boxed контейнер</label>
        </div>

        <div class="field">
          <label>Фон секции</label>
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
          <label><input id="sec_border" type="checkbox"> Граница</label>
        </div>

        <div class="field">
          <label>Скругление (0..40)</label>
          <input id="sec_radius" class="input" type="number" min="0" max="40" value="0" />
        </div>
      </div>
    `,
    buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
    onOk: function (mb) {
      const boxed = document.getElementById('sec_boxed')?.checked ? '1' : '0';
      const background = (document.getElementById('sec_bg')?.value || '#FFFFFF').trim();
      const paddingTop = parseInt(document.getElementById('sec_pt')?.value || '32', 10);
      const paddingBottom = parseInt(document.getElementById('sec_pb')?.value || '32', 10);
      const border = document.getElementById('sec_border')?.checked ? '1' : '0';
      const radius = parseInt(document.getElementById('sec_radius')?.value || '0', 10);

      api('block.create', {
        pageId,
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
          mb.close();
          loadBlocks();
        })
        .catch(() => notify('Ошибка block.create (section)'));
    }
  });
}

5. Привязать кнопку

Там, где у тебя внизу идут обработчики вроде:

btnAddText?.addEventListener('click', addTextBlock);
btnAddImage?.addEventListener('click', addImageBlock);
...
btnAddCards?.addEventListener('click', addCardsBlock);

добавь:

btnAddSection?.addEventListener('click', addSectionBlock);


---

Что ещё нужно для полноценной работы

Чтобы section не просто создавался, но и был виден в списке блоков, нужно добавить его в renderBlocks(blocks).

6. Добавить рендер section в списке блоков

Внутри renderBlocks(blocks) перед TEXT или сразу после начала цикла вставь:

if (type === 'section') {
  const c = (b.content && typeof b.content === 'object') ? b.content : {};
  const boxed = !!c.boxed;
  const background = c.background || '#FFFFFF';
  const paddingTop = parseInt(c.paddingTop || 32, 10);
  const paddingBottom = parseInt(c.paddingBottom || 32, 10);
  const border = !!c.border;
  const radius = parseInt(c.radius || 0, 10);

  return `
    <div class="block" style="border:1px solid #cbd5e1;background:#f8fafc;">
      <div class="row">
        <div><b>#${id}</b> <span class="muted">(section | sort ${sort})</span></div>
        <div class="btns">
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="up">↑</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-move-block-id="${id}" data-move-dir="down">↓</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-section-id="${id}">Редактировать</button>
          <button class="ui-btn ui-btn-light ui-btn-xs" data-dup-block-id="${id}">Дублировать</button>
          <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
        </div>
      </div>

      <div class="muted" style="margin-top:8px;">
        boxed: <b>${boxed ? 'yes' : 'no'}</b> ·
        bg: <code>${BX.util.htmlspecialchars(background)}</code> ·
        pt: <b>${paddingTop}</b> ·
        pb: <b>${paddingBottom}</b> ·
        border: <b>${border ? 'yes' : 'no'}</b> ·
        radius: <b>${radius}</b>
      </div>
    </div>
  `;
}

7. Добавить редактирование section

Рядом с другими edit...Block функциями вставь:

function editSectionBlock(id) {
  api('block.list', { pageId }).then(res => {
    if (!res || res.ok !== true) return;
    const blk = (res.blocks || []).find(x => parseInt(x.id, 10) === id);
    const cur = (blk && blk.content && typeof blk.content === 'object') ? blk.content : {};

    BX.UI.Dialogs.MessageBox.show({
      title: 'Редактировать Section #' + id,
      message: `
        <div>
          <div class="field">
            <label><input id="sec_boxed_e" type="checkbox" ${cur.boxed ? 'checked' : ''}> Boxed контейнер</label>
          </div>

          <div class="field">
            <label>Фон секции</label>
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
            <label><input id="sec_border_e" type="checkbox" ${cur.border ? 'checked' : ''}> Граница</label>
          </div>

          <div class="field">
            <label>Скругление (0..40)</label>
            <input id="sec_radius_e" class="input" type="number" min="0" max="40" value="${parseInt(cur.radius || 0, 10)}" />
          </div>
        </div>
      `,
      buttons: BX.UI.Dialogs.MessageBoxButtons.OK_CANCEL,
      onOk: function (mb) {
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
            mb.close();
            loadBlocks();
          })
          .catch(() => notify('Ошибка block.update (section)'));
      }
    });
  });
}

8. Подключить обработчик клика

В общем document.addEventListener('click', ...), где у тебя уже есть проверки типа data-edit-text-id, data-edit-image-id и т.д., добавь:

const editSectionBtn = e.target.closest('[data-edit-section-id]');
if (editSectionBtn) {
  editSectionBlock(parseInt(editSectionBtn.getAttribute('data-edit-section-id'), 10));
  return;
}


---

Вывод

После этих правок в editor.php у тебя появится:

кнопка + Section

создание секции

отображение секции в списке блоков

редактирование секции


Но ещё раз: проверь api.php, потому что в архиве он пока не принимает section. Без этого кнопка будет показываться, но сохранение не сработает.

Если хочешь, следующим сообщением я соберу тебе полностью готовый editor.php целиком под текущую структуру.