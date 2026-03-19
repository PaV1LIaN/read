Нашёл конкретные ошибки.

Почему сейчас не работает

1. section не участвует в drag&drop

У тебя ветка if (type === 'section') возвращает свой собственный HTML, а не buildBlockShell(...).

Из-за этого у section нет:

data-block-id

dragHandle

общей шапки блока

нормальной зоны перетаскивания


То есть section сейчас вообще выпадает из механики DnD.


---

2. Порядок не сохраняется после перетаскивания

saveBlockOrder() собирает порядок так:

const ids = Array.from(blocksBox.querySelectorAll('.block[data-block-id]'))

А section не попадает в этот список, потому что у неё нет data-block-id.

В итоге сервер получает неполный список id, и block.reorder не может сохранить корректный порядок.


---

3. У обычных блоков ты завёл wrap = wrapBlockMeta(), но не передаёшь его в buildBlockShell(...)

То есть:

логика “внутри секции” вычисляется

но в рендер не попадает


Это не ломает dnd напрямую, но показывает, что часть правки осталась недоделанной.


---

Что сделать сейчас

Ниже даю точечные правки.


---

1. Замени ветку section целиком

Найди в renderBlocks(blocks):

if (type === 'section') {
  currentSectionId = id;

  const c = (b.content && typeof b.content === 'object') ? b.content : {};
  ...
  return `
    <div class="block blockSection">
      ...
    </div>
  `;
}

Замени всю ветку целиком на это:

if (type === 'section') {
  currentSectionId = id;

  const c = (b.content && typeof b.content === 'object') ? b.content : {};
  const boxed = !!c.boxed;
  const background = c.background || '#FFFFFF';
  const paddingTop = parseInt(c.paddingTop || 32, 10);
  const paddingBottom = parseInt(c.paddingBottom || 32, 10);
  const border = !!c.border;
  const radius = parseInt(c.radius || 0, 10);

  return buildBlockShell(
    id,
    type,
    sort,
    `
      <div class="blockSectionGrid">
        <div class="blockSectionItem">
          <div class="blockSectionLabel">Контейнер</div>
          <div class="blockSectionValue">${boxed ? 'Boxed' : 'Full width'}</div>
        </div>

        <div class="blockSectionItem">
          <div class="blockSectionLabel">Фон</div>
          <div class="blockSectionValue">
            <span style="display:inline-block;width:12px;height:12px;border-radius:3px;background:${BX.util.htmlspecialchars(background)};border:1px solid #cbd5e1;vertical-align:-1px;margin-right:6px;"></span>
            ${BX.util.htmlspecialchars(background)}
          </div>
        </div>

        <div class="blockSectionItem">
          <div class="blockSectionLabel">Отступы</div>
          <div class="blockSectionValue">top ${paddingTop}px / bottom ${paddingBottom}px</div>
        </div>

        <div class="blockSectionItem">
          <div class="blockSectionLabel">Граница</div>
          <div class="blockSectionValue">${border ? 'Да' : 'Нет'}</div>
        </div>

        <div class="blockSectionItem">
          <div class="blockSectionLabel">Скругление</div>
          <div class="blockSectionValue">${radius}px</div>
        </div>
      </div>

      <div class="btns" style="margin-top:10px;">
        <button class="ui-btn ui-btn-success ui-btn-xs" data-add-heading-after-section-id="${id}">+ Heading</button>
        <button class="ui-btn ui-btn-success ui-btn-xs" data-add-text-after-section-id="${id}">+ Text</button>
        <button class="ui-btn ui-btn-success ui-btn-xs" data-add-button-after-section-id="${id}">+ Button</button>
        <button class="ui-btn ui-btn-success ui-btn-xs" data-add-cards-after-section-id="${id}">+ Cards</button>
      </div>
    `,
    `
      ${commonBtns}
      <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-section-id="${id}">Редактировать</button>
      <button class="ui-btn ui-btn-light ui-btn-xs" data-dup-block-id="${id}">Дублировать</button>
      <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
    `,
    '',
    'blockSection'
  );
}


---

2. Исправь обычные блоки: передавай wrap в buildBlockShell(...)

Сейчас у тебя есть, например, text:

const wrap = wrapBlockMeta();
return buildBlockShell(
  id, type, sort,
  ...,
  ...
);

Нужно сделать так:

Для каждого обычного блока добавь в конец buildBlockShell(...):

wrap.extraClass,
wrap.sectionMarkHtml


---

Пример для text

Было:

const wrap = wrapBlockMeta();
return buildBlockShell(
  id, type, sort,
  `<pre>${BX.util.htmlspecialchars(text)}</pre>`,
  `
    ${commonBtns}
    <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
    <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
  `
);

Должно стать:

const wrap = wrapBlockMeta();
return buildBlockShell(
  id, type, sort,
  `<pre>${BX.util.htmlspecialchars(text)}</pre>`,
  `
    ${commonBtns}
    <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
    <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
  `,
  '',
  wrap.extraClass,
  wrap.sectionMarkHtml
);


---

То же самое сделай для:

text

image

button

heading

columns2

gallery

spacer

card

cards

fallback unknown type



---

3. Ничего не меняй в saveBlockOrder()

После фикса section через buildBlockShell(...) этот код начнёт работать нормально:

const ids = Array.from(blocksBox.querySelectorAll('.block[data-block-id]'))

Потому что теперь section тоже станет .block[data-block-id].


---

4. Почему после этого всё должно заработать

Теперь:

section получит drag handle

section будет draggable

section будет входить в saveBlockOrder()

сервер получит полный список id

порядок начнёт сохраняться



---

Что проверить после правки

1. У section появился значок перетаскивания ⋮⋮

Если не появился — ветка section всё ещё рендерится не через buildBlockShell.

2. Перетаскивание section

Попробуй перетащить section вверх/вниз.

3. Перетаскивание обычных блоков

Проверь, что после drag порядок сохраняется, а после loadBlocks() не откатывается.

4. Если всё ещё не сохраняется

Тогда пришли текст ответа block.reorder из Network/console, потому что тогда уже надо смотреть, что именно сервер возвращает.


---

Если хочешь, я следующим сообщением могу дать тебе готовые куски именно для text/image/button/heading/..., чтобы ты не правил 10 блоков вручную.