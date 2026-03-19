Проблема найдена.

Сейчас перетаскивание ломается из-за того, что после добавления blockInSection у тебя DOM стал таким:

section → это .block

обычный блок внутри секции → это .blockInSection > .block


А drag&drop работает только с .block[data-block-id] и при dragover делает:

block.parentNode.insertBefore(draggedEl, block)

То есть перетаскивает элемент только внутри его текущего parentNode.
Из-за этого блоки в секции и вне секции живут в разных контейнерах, и сортировка начинает вести себя криво.

Как починить правильно

Нужно убрать внешнюю обёртку .blockInSection и вместо неё вешать “внутри секции” прямо на сам .block.

Это самый чистый фикс.


---

Что поменять

1. В buildBlockShell(...) добавь ещё один параметр

Найди:

function buildBlockShell(id, type, sort, bodyHtml, buttonsHtml, extraMetaHtml = '') {

Замени на:

function buildBlockShell(id, type, sort, bodyHtml, buttonsHtml, extraMetaHtml = '', extraClass = '', sectionMarkHtml = '') {


---

2. Внутри buildBlockShell(...) замени открывающий <div class="block ...">

Найди:

<div class="block ${isCollapsed ? 'blockCollapsed' : ''}" data-type="${BX.util.htmlspecialchars(type)}" data-block-id="${id}">

Замени на:

<div class="block ${isCollapsed ? 'blockCollapsed' : ''} ${extraClass}" data-type="${BX.util.htmlspecialchars(type)}" data-block-id="${id}">


---

3. В blockBody выведи метку секции

Найди:

<div class="blockBody">
  ${bodyHtml}
</div>

Замени на:

<div class="blockBody">
  ${sectionMarkHtml}
  ${bodyHtml}
</div>


---

4. Полностью замени wrapBlockHtml(...)

Сейчас у тебя:

function wrapBlockHtml(innerHtml) {
  if (!currentSectionId) return innerHtml;

  return `
    <div class="blockInSection">
      <div class="blockSectionDivider">inside section #${currentSectionId}</div>
      ${innerHtml}
    </div>
  `;
}

Замени на это:

function wrapBlockMeta() {
  if (!currentSectionId) {
    return {
      extraClass: '',
      sectionMarkHtml: ''
    };
  }

  return {
    extraClass: 'blockInSection',
    sectionMarkHtml: `<div class="blockSectionDivider">inside section #${currentSectionId}</div>`
  };
}


---

5. Во всех обычных блоках убери wrapBlockHtml(...)

То есть вместо:

return wrapBlockHtml(buildBlockShell(
  ...
));

нужно делать так:

const wrap = wrapBlockMeta();
return buildBlockShell(
  id, type, sort,
  ...,
  ...,
  ...,
  wrap.extraClass,
  wrap.sectionMarkHtml
);


---

Покажу на примере

Было у text

return wrapBlockHtml(buildBlockShell(
  id, type, sort,
  `<pre>${BX.util.htmlspecialchars(text)}</pre>`,
  `
    ${commonBtns}
    <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
    <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
  `
));

Должно стать

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

То же самое сделать для:

text

image

button

heading

columns2

gallery

spacer

card

cards

unknown type



---

6. CSS почти можно оставить, но поправь blockInSection

Найди:

.blockInSection{
  margin-left:18px;
  padding-left:12px;
  border-left:2px solid #dbeafe;
}

Замени на:

.blockInSection{
  margin-left:18px;
  border-left:2px solid #dbeafe;
}

Потому что теперь blockInSection висит на самой .block, а не на внешней обёртке.


---

Почему это чинит drag&drop

Теперь все draggable-элементы снова будут одного типа:

<div class="block ..." data-block-id="...">

И у них будет один и тот же уровень DOM в blocksBox.innerHTML, без вложенных внешних контейнеров.

Значит:

dragover

insertBefore

saveBlockOrder

block.reorder


снова начнут работать корректно.


---

Что проверить после правок

1. Перетаскивание обычных блоков вверх/вниз.


2. Перетаскивание section.


3. Перетаскивание блока из-под одной секции к другой.


4. После loadBlocks() порядок сохраняется правильно.



Если хочешь, следующим сообщением я могу прямо по твоему текущему renderBlocks() расписать готовые замены для text/image/button/..., чтобы ты не правил каждый блок вручную.