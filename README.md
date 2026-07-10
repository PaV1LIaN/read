Отлично — серверная часть работает полностью:

page.list возвращает страницы;

block.list возвращает блоки страницы;

права доступа есть;

проблема теперь находится в JavaScript редактора.


Скорее всего, в /local/sitebuilder/editor.php код ищет данные так:

result.data.pages

и:

result.data.blocks

Но API возвращает их напрямую:

result.pages
result.blocks

В editor.php найди обращения к data.pages и data.blocks и замени.

Было:

const pages = result.data?.pages || [];

Стало:

const pages = Array.isArray(result.pages)
  ? result.pages
  : [];

Было:

const blocks = result.data?.blocks || [];

Стало:

const blocks = Array.isArray(result.blocks)
  ? result.blocks
  : [];

Также проверь похожие конструкции:

response.data.pages
response.data.blocks
json.data.pages
json.data.blocks

Они должны соответствовать реальному ответу:

response.pages
response.blocks
json.pages
json.blocks

Для временной диагностики добавь после загрузки страниц:

console.log('PAGE LIST RESULT:', result);
console.log('PAGES:', result.pages);

А после загрузки блоков:

console.log('BLOCK LIST RESULT:', result);
console.log('BLOCKS:', result.blocks);

После изменения:

1. Сохрани /local/sitebuilder/editor.php.


2. Открой редактор.


3. Выполни жёсткое обновление страницы: Ctrl + F5.


4. Выбери страницу «Тест».


5. В дереве должны появиться страницы «Диск» и «Тест», а на холсте — блоки heading и text.



То есть API больше исправлять не нужно. Сейчас исправляем получение результата в editor.php.