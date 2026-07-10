(async function () {
  const sessidData = await fetch(
    '/local/sitebuilder/api/get_sessid.php',
    {
      credentials: 'same-origin',
      cache: 'no-store'
    }
  ).then(response => response.json());

  const fd = new FormData();

  fd.append('action', 'page.list');
  fd.append('sessid', sessidData.sessid);
  fd.append('siteId', '13');

  const response = await fetch(
    '/local/sitebuilder/api/index.php',
    {
      method: 'POST',
      body: fd,
      credentials: 'same-origin',
      cache: 'no-store'
    }
  );

  const result = await response.json();

  console.log(
    result.data?.pages?.map(page => ({
      id: page.id,
      title: page.title,
      parentId: page.parentId,
      navigationOnly: page.navigationOnly,
      access: page.access
    }))
  );

  return result;
})();
Promise {<pending>}
VM719:2  GET https://portal24.itsnn.ru/local/sitebuilder/api/get_sessid.php 404 (Not Found)
(anonymous) @ VM719:2
(anonymous) @ VM719:39
VM720:1 Uncaught (in promise) SyntaxError: Unexpected token 'F', "File not found.
" is not valid JSON
