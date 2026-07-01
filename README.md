Теперь ввыводит array, где должен быть текст

(async function () {
  const pageId = window.SB_PAGE_ID || window.pageId || null;

  console.log('pageId:', pageId);

  if (!pageId) {
    console.warn('Не нашёл pageId глобально. Открой Network и посмотри ответ page.get/block.list.');
  }

  console.log('Ищи блок type=text в данных страницы/блоков.');
})();
VM174:4 pageId: null
VM174:7 Не нашёл pageId глобально. Открой Network и посмотри ответ page.get/block.list.
(анонимный) @ VM174:7
(анонимный) @ VM174:11
VM174:10 Ищи блок type=text в данных страницы/блоков.
