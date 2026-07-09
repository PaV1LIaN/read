Да, сюда. Только в твоей архитектуре лучше не в actions/page_access/..., а через один handler:

/local/sitebuilder/api/handlers/page_access.php

А в index.php добавить маршрутизацию.


---

1. В index.php добавь блок

Файл:

/local/sitebuilder/api/index.php

Лучше вставь после блока page.*, вот здесь:

if (
    $action === 'page.list' ||
    $action === 'page.create' ||
    $action === 'page.delete' ||
    $action === 'page.duplicate' ||
    $action === 'page.updateMeta' ||
    $action === 'page.setStatus' ||
    $action === 'page.setParent' ||
    $action === 'page.move'
) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

Сразу после него вставь:

if (
    $action === 'pageAccess.list' ||
    $action === 'pageAccess.save' ||
    $action === 'pageAccess.delete'
) {
    require __DIR__ . '/handlers/page_access.php';
    exit;
}

Итог:

if (
    $action === 'page.list' ||
    $action === 'page.create' ||
    $action === 'page.delete' ||
    $action === 'page.duplicate' ||
    $action === 'page.updateMeta' ||
    $action === 'page.setStatus' ||
    $action === 'page.setParent' ||
    $action === 'page.move'
) {
    require __DIR__ . '/handlers/page.php';
    exit;
}

if (
    $action === 'pageAccess.list' ||
    $action === 'pageAccess.save' ||
    $action === 'pageAccess.delete'
) {
    require __DIR__ . '/handlers/page_access.php';
    exit;
}


---

2. Важно

У тебя index.php берёт action так:

$action = (string)($_POST['action'] ?? '');

Значит вот такой запрос не сработает:

fetch('/local/sitebuilder/api/?action=pageAccess.save')

Нужно отправлять action именно в POST.

То есть тесты потом будем делать так:

const fd = new FormData();
fd.append('action', 'pageAccess.save');
fd.append('sessid', BX.bitrix_sessid());
fd.append('siteId', siteId);
fd.append('pageId', pageId);
fd.append('accessCode', 'U' + userId);
fd.append('canView', '1');
fd.append('canEdit', '1');
fd.append('includeChildren', '1');

const res = await fetch('/local/sitebuilder/api/index.php', {
  method: 'POST',
  body: fd
});

console.log(await res.json());


---

3. Следующий файл

Теперь создай:

/local/sitebuilder/api/handlers/page_access.php

И дальше уже туда вставим обработку pageAccess.list/save/delete.