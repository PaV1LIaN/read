Ошибка понятная: в page.php на строке 176 старая логика вызывает функцию так:

sb_page_is_descendant($pages, $id, $parentId)

А сейчас функция в helpers.php ждёт первым аргументом ID сайта, а не массив страниц:

sb_page_is_descendant(int $siteId, ...)

То есть нужно заменить вызов.

Что исправить

Открой файл:

/local/sitebuilder/api/handlers/page.php

Найди примерно такой кусок в обработчике page.updateMeta:

if ($parentId > 0) {
    if ($parentId === $id) {
        sb_json_error('PAGE_CANNOT_BE_PARENT_OF_ITSELF', 422);
    }

    if (sb_page_is_descendant($pages, $id, $parentId)) {
        sb_json_error('PAGE_CANNOT_BE_MOVED_TO_OWN_CHILD', 422);
    }
}

Замени на:

if ($parentId > 0) {
    if ($parentId === $id) {
        sb_json_error('PAGE_CANNOT_BE_PARENT_OF_ITSELF', 422);
    }

    if (sb_page_is_descendant($siteId, $id, $parentId)) {
        sb_json_error('PAGE_CANNOT_BE_MOVED_TO_OWN_CHILD', 422);
    }
}

Если рядом нет $siteId

Перед этим блоком должен быть получен текущий объект страницы. Проверь, чтобы выше было что-то такое:

$page = sb_get_page_by_id($id);

if (!$page) {
    sb_json_error('PAGE_NOT_FOUND', 404);
}

$siteId = (int)($page['siteId'] ?? 0);

if ($siteId <= 0) {
    sb_json_error('SITE_ID_NOT_FOUND', 422);
}

Если $page уже называется иначе, например $currentPage, тогда так:

$siteId = (int)($currentPage['siteId'] ?? 0);

Главное — в строке 176 должно стать именно:

sb_page_is_descendant($siteId, $id, $parentId)

После этого смена родительской страницы через “Сохранить страницу” должна заработать.