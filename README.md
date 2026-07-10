Причина в текущей логике: у пользователя U99, скорее всего, уже есть глобальная роль в:

sitebuilder.access

А в page.php сейчас написано: если есть глобальное право просмотра сайта, показать все страницы.

То есть право на страницу сейчас работает как дополнительное, а не как ограничение.

Проверь запись:

SELECT *
FROM sitebuilder.access
WHERE site_id = 13
  AND access_code = 'U99';

Скорее всего, там будет роль VIEWER.

Исправляем правило так:

OWNER, ADMIN, EDITOR видят все страницы;

VIEWER без индивидуальных прав видит все страницы;

VIEWER, которому добавлены права в page_access, видит только разрешённые страницы;

пользователь без глобальной роли, но с page_access, тоже видит только разрешённые страницы.


1. Замени функцию фильтрации

В файле:

/local/sitebuilder/api/handlers/page.php

найди функцию:

sb_page_handler_filter_visible_pages

И замени её полностью:

if (!function_exists('sb_page_handler_filter_visible_pages')) {
    function sb_page_handler_filter_visible_pages(
        array $pages,
        int $siteId,
        int $userId
    ): array {
        $hasGlobalView = sb_page_handler_has_global_view(
            $siteId,
            $userId
        );

        $hasGlobalEdit = sb_page_handler_has_global_edit(
            $siteId,
            $userId
        );

        $hasPageAccess = PageAccessService::hasAnyPageAccess(
            $siteId,
            $userId
        );

        /*
         * Администратор Битрикса, OWNER, ADMIN и EDITOR
         * видят все страницы сайта.
         */
        if ($hasGlobalEdit) {
            return array_values(array_map(
                static function ($page) use ($siteId, $userId) {
                    $page = sb_normalize_page_record($page);

                    return sb_page_handler_add_access_info(
                        $page,
                        $siteId,
                        $userId
                    );
                },
                $pages
            ));
        }

        /*
         * Старое поведение:
         * глобальный VIEWER без индивидуальных правил
         * видит весь сайт.
         */
        if ($hasGlobalView && !$hasPageAccess) {
            return array_values(array_map(
                static function ($page) use ($siteId, $userId) {
                    $page = sb_normalize_page_record($page);

                    return sb_page_handler_add_access_info(
                        $page,
                        $siteId,
                        $userId
                    );
                },
                $pages
            ));
        }

        /*
         * Если существуют индивидуальные права страниц,
         * фильтруем только по sitebuilder.page_access.
         *
         * Здесь нельзя использовать canViewPage(),
         * потому что глобальный VIEWER через него снова
         * получит доступ ко всем страницам.
         */
        $accessCode = PageAccessRepository::userAccessCode($userId);

        $pagesById = [];

        foreach ($pages as $page) {
            $pageId = (int)($page['id'] ?? 0);

            if ($pageId > 0) {
                $pagesById[$pageId] = $page;
            }
        }

        $includedIds = [];
        $permissionsByPageId = [];

        foreach ($pages as $page) {
            $pageId = (int)($page['id'] ?? 0);

            if ($pageId <= 0) {
                continue;
            }

            $canView = PageAccessRepository::hasPagePermission(
                $siteId,
                $pageId,
                $accessCode,
                'view'
            );

            $canEdit = PageAccessRepository::hasPagePermission(
                $siteId,
                $pageId,
                $accessCode,
                'edit'
            );

            if (!$canView && !$canEdit) {
                continue;
            }

            $includedIds[$pageId] = true;

            $permissionsByPageId[$pageId] = [
                'canView' => $canView || $canEdit,
                'canEdit' => $canEdit,
            ];
        }

        /*
         * Добавляем родителей доступных страниц,
         * чтобы не ломалась древовидная структура.
         *
         * Такие родители будут navigationOnly.
         */
        foreach (array_keys($includedIds) as $pageId) {
            $currentId = (int)$pageId;
            $safety = 0;

            while ($currentId > 0 && $safety < 1000) {
                $currentPage = $pagesById[$currentId] ?? null;

                if (!$currentPage) {
                    break;
                }

                $parentId = (int)($currentPage['parentId'] ?? 0);

                if ($parentId <= 0) {
                    break;
                }

                $includedIds[$parentId] = true;

                if (!isset($permissionsByPageId[$parentId])) {
                    $permissionsByPageId[$parentId] = [
                        'canView' => false,
                        'canEdit' => false,
                    ];
                }

                $currentId = $parentId;
                $safety++;
            }
        }

        $result = [];

        foreach ($pages as $page) {
            $pageId = (int)($page['id'] ?? 0);

            if (!isset($includedIds[$pageId])) {
                continue;
            }

            $page = sb_normalize_page_record($page);

            $permission = $permissionsByPageId[$pageId] ?? [
                'canView' => false,
                'canEdit' => false,
            ];

            $page['access'] = [
                'canView' => (bool)$permission['canView'],
                'canEdit' => (bool)$permission['canEdit'],
            ];

            $page['navigationOnly'] = !$permission['canView'];

            $result[] = $page;
        }

        return array_values($result);
    }
}

2. Сам page.list менять не нужно

В нём уже вызывается:

$pages = sb_page_handler_filter_visible_pages(
    $pages,
    $siteId,
    $currentUserId
);

После замены функции пользователь 99 должен увидеть:

страницу 14;

дочерние страницы страницы 14, потому что было выдано includeChildren = true;

родителей страницы 14 только как узлы дерева с navigationOnly: true;

остальные страницы не должны приходить в ответе API.


3. Проверка через консоль под пользователем 99

На странице редактора выполни:

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

Ключевая ошибка была именно в этой проверке:

if (sb_page_handler_has_global_view($siteId, $userId)) {
    return все страницы;
}

Теперь глобальный VIEWER с индивидуальными правами будет ограничен таблицей page_access.