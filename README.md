Давай. Сделаем, чтобы в editor.php было видно: какие блоки относятся к какой section.

Идея простая:

SECTION остаётся отдельной заметной карточкой

все блоки после неё до следующей SECTION получают:

левую линию

небольшой сдвиг

пометку, что они внутри секции



Это сильно улучшит читаемость.

Сейчас загруженный архив у меня уже не открывается, поэтому дам точечные правки, которые вставляются в твой текущий editor.php.

1. Добавь CSS для блоков внутри section

В <style> editor.php добавь:

.blockInSection{
  margin-left:18px;
  padding-left:12px;
  border-left:2px solid #dbeafe;
}

.blockInSection .row{
  position:relative;
}

.blockInSectionTag{
  display:inline-flex;
  align-items:center;
  padding:2px 8px;
  border-radius:999px;
  font-size:11px;
  font-weight:600;
  background:#eff6ff;
  color:#1d4ed8;
  border:1px solid #bfdbfe;
  margin-left:8px;
}

.blockSectionDivider{
  margin-top:8px;
  margin-bottom:4px;
  font-size:11px;
  color:#64748b;
  text-transform:uppercase;
  letter-spacing:.04em;
}


---

2. В renderBlocks(blocks) добавь флаг текущей секции

В начале renderBlocks(blocks) найди что-то вроде:

function renderBlocks(blocks) {
  if (!blocks || !blocks.length) {
    ...
  }

  return blocks.map(b => {

И перед return blocks.map(...) добавь:

let currentSectionId = null;

То есть должно получиться примерно так:

function renderBlocks(blocks) {
  if (!blocks || !blocks.length) {
    blocksBox.innerHTML = '<div class="muted">Нет блоков</div>';
    return;
  }

  let currentSectionId = null;

  return blocks.map(b => {


---

3. Помечай section как начало группы

Внутри renderBlocks(blocks) в ветке:

if (type === 'section') {

в самое начало добавь:

currentSectionId = id;

То есть:

if (type === 'section') {
  currentSectionId = id;

  const c = ...


---

4. Для обычных блоков добавь обёртку “внутри секции”

В каждой ветке обычного блока (text, image, button, heading, columns2, gallery, spacer, card, cards) у тебя сейчас, скорее всего, возвращается что-то вроде:

return `
  <div class="block">
    ...
  </div>
`;

Нужно сделать общий helper в начале renderBlocks(blocks):

function wrapBlockHtml(innerHtml) {
  if (!currentSectionId) return innerHtml;

  return `
    <div class="blockInSection">
      <div class="blockSectionDivider">inside section #${currentSectionId}</div>
      ${innerHtml}
    </div>
  `;
}

Вставь его сразу после:

let currentSectionId = null;

То есть так:

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


---

5. Оберни обычные блоки через wrapBlockHtml(...)

Теперь в каждом обычном блоке вместо:

return `
  <div class="block">
    ...
  </div>
`;

делай:

return wrapBlockHtml(`
  <div class="block">
    ...
  </div>
`);

Пример для text

Было:

return `
  <div class="block">
    <div class="row">
      ...
    </div>
  </div>
`;

Станет:

return wrapBlockHtml(`
  <div class="block">
    <div class="row">
      ...
    </div>
  </div>
`);

То же самое для:

image

button

heading

columns2

gallery

spacer

card

cards



---

6. Не сбрасывай section до следующей section

Тут важно: currentSectionId должен оставаться активным, пока не встретится следующая section.

То есть:

встретили section #12

все следующие блоки идут как inside section #12

встретили section #18

теперь все следующие идут как inside section #18


Дополнительно ничего сбрасывать не нужно.


---

Что получится

В редакторе будет видно:

где начинается новая SECTION

какие блоки относятся к ней

какая именно секция их оборачивает


Это уже сильно улучшит UX.


---

Что бы я делал следующим шагом

После этого логично сделать у section быстрые кнопки:

+ Heading

+ Text

+ Button

+ Cards


чтобы можно было сразу наполнять секцию контентом.