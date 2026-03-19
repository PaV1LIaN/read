Да, по твоему текущему editor.php скажу точно, куда вставлять.

У тебя уже:

CSS для .blockInSection есть

section в renderBlocks() есть


Но сейчас обычные блоки не оборачиваются как “внутри section”.
Нужно добавить только 3 точечные правки.


---

1. В renderBlocks(blocks) добавь currentSectionId

Найди

В renderBlocks(blocks) у тебя сейчас есть:

blocksBox.innerHTML = filteredBlocks.map(b => {
  const type = b.type || '';
  const sort = b.sort;
  const id = b.id;

Замени на

let currentSectionId = null;

blocksBox.innerHTML = filteredBlocks.map(b => {
  const type = b.type || '';
  const sort = b.sort;
  const id = b.id;


---

2. Добавь helper wrapBlockHtml(...)

Куда вставить

Сразу после строки:

let currentSectionId = null;

вставь это:

function wrapBlockHtml(innerHtml) {
  if (!currentSectionId) return innerHtml;

  return `
    <div class="blockInSection">
      <div class="blockSectionDivider">inside section #${currentSectionId}</div>
      ${innerHtml}
    </div>
  `;
}

То есть начало куска должно стать таким:

let currentSectionId = null;

function wrapBlockHtml(innerHtml) {
  if (!currentSectionId) return innerHtml;

  return `
    <div class="blockInSection">
      <div class="blockSectionDivider">inside section #${currentSectionId}</div>
      ${innerHtml}
    </div>
  `;
}

blocksBox.innerHTML = filteredBlocks.map(b => {


---

3. В ветке section запоминай текущую секцию

Найди

У тебя уже есть:

if (type === 'section') {
  const c = (b.content && typeof b.content === 'object') ? b.content : {};

Замени начало на:

if (type === 'section') {
  currentSectionId = id;

  const c = (b.content && typeof b.content === 'object') ? b.content : {};


---

4. Оберни обычные блоки через wrapBlockHtml(...)

Это главное.

Сейчас у тебя обычные блоки возвращаются так:

return buildBlockShell(
  ...
);

Нужно заменить на:

return wrapBlockHtml(buildBlockShell(
  ...
));


---

Что именно менять

Было у text

return buildBlockShell(
  id, type, sort,
  `<pre>${BX.util.htmlspecialchars(text)}</pre>`,
  `
    ${commonBtns}
    <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
    <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
  `
);

Должно стать

return wrapBlockHtml(buildBlockShell(
  id, type, sort,
  `<pre>${BX.util.htmlspecialchars(text)}</pre>`,
  `
    ${commonBtns}
    <button class="ui-btn ui-btn-light ui-btn-xs" data-edit-text-id="${id}">Редактировать</button>
    <button class="ui-btn ui-btn-danger ui-btn-xs" data-del-block-id="${id}">Удалить</button>
  `
));


---

То же самое сделать для всех обычных типов:

text

image

button

heading

columns2

gallery

spacer

card

cards


Принцип один и тот же

Вместо:

return buildBlockShell(...)

делай:

return wrapBlockHtml(buildBlockShell(...))


---

Что НЕ трогать

Ветку if (type === 'section') { ... } не оборачивай через wrapBlockHtml.

section должна оставаться самостоятельной карточкой.


---

Что получится

Теперь логика будет такой:

встретили section #15

все следующие обычные блоки будут показываться как:


inside section #15
[блок]

встретили следующую section #21

дальше уже пойдут как inside section #21



---

Если хочешь совсем быстро

Самые важные места в твоём текущем файле:

renderBlocks(blocks) начинается примерно там, где у тебя уже видно const filteredBlocks...

ветка if (type === 'section') { у тебя уже есть

дальше все return buildBlockShell(...) ниже по функции надо просто обернуть



---

Если покажешь мне кусок renderBlocks() от if (type === 'text') до конца функции, я прямо по твоему коду отмечу, какие конкретно return buildBlockShell заменить.