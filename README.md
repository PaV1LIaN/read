page.php в целом уже правильно применяет точечные права:

page.list фильтрует страницы;

родители доступных страниц добавляются как navigationOnly;

page.create проверяет право редактирования родителя;

изменение, публикация и удаление требуют canEdit;

при удалении ветки проверяется каждая дочерняя страница;

includeChildren учитывается через PageAccessRepository.


Но перед тестированием нужно исправить два обхода логики прав.

1. Глобальный VIEWER не должен терять доступ ко всему сайту

Сейчас глобальный VIEWER, которому дополнительно выдали право на одну страницу, начинает видеть только страницы из page_access.

Это противоречит выбранной модели:

глобальная роль ИЛИ точечное право

Полностью замени функцию sb_page_handler_filter_visible_pages():

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

        /*
         * Любая глобальная роль с правом просмотра
         * сохраняет доступ ко всем страницам.
         *
         * Индивидуальные правила при этом могут дополнительно
         * дать canEdit для отдельных страниц.
         */
        if ($hasGlobalView) {
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
         * Пользователь без глобальной роли получает только
         * страницы, разрешённые через page_access.
         */
        $accessCode = PageAccessRepository::userAccessCode(
            $userId
        );

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
         * Добавляем родителей разрешённых страниц,
         * чтобы сохранить дерево навигации.
         */
        foreach (array_keys($includedIds) as $pageId) {
            $currentId = (int)$pageId;
            $visited = [];

            while ($currentId > 0) {
                if (isset($visited[$currentId])) {
                    break;
                }

                $visited[$currentId] = true;

                $currentPage = $pagesById[$currentId] ?? null;

                if (!$currentPage) {
                    break;
                }

                $parentId = (int)(
                    $currentPage['parentId'] ?? 0
                );

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

            $page['navigationOnly'] =
                !$permission['canView'];

            $result[] = $page;
        }

        return array_values($result);
    }
}

После этой правки:

глобальный VIEWER видит все страницы;

глобальный VIEWER с точечным canEdit редактирует выбранные страницы;

пользователь без глобальной роли видит только назначенные страницы;

глобальные EDITOR, ADMIN, OWNER продолжают видеть весь сайт.


2. Закрой обход создания корневых страниц через дублирование

Сейчас пользователь с точечным canEdit корневой страницы может выполнить page.duplicate и создать новую корневую страницу, хотя page.create запрещает ему создавать корневые страницы.

В обработчике:

if ($action === 'page.duplicate') {

после:

$hasGlobalEdit = sb_page_handler_has_global_edit(
    $siteId,
    $currentUserId
);

вставь:

$sourceParentId = (int)($source['parentId'] ?? 0);

/*
 * Дублирование создаёт новую страницу рядом с исходной.
 * Поэтому применяем те же ограничения, что и при page.create.
 */
if (!$hasGlobalEdit) {
    if ($sourceParentId <= 0) {
        sb_json_error(
            'ROOT_PAGE_CREATE_ACCESS_DENIED',
            403
        );
    }

    if (
        !PageAccessService::canEditPage(
            $siteId,
            $sourceParentId,
            $currentUserId
        )
    ) {
        sb_json_error(
            'PARENT_PAGE_EDIT_ACCESS_DENIED',
            403,
            [
                'parentId' => $sourceParentId,
            ]
        );
    }
}

Ниже замени:

'parentId' => (int)($source['parentId'] ?? 0),

на:

'parentId' => $sourceParentId,

Ещё один важный момент

Сейчас пользователь с точечным canEdit может сам назначать права другим пользователям, потому что в page_access.php есть:

return PageAccessService::canEditPage(
    $siteId,
    $pageId,
    $userId
);

Для серьёзного проекта безопаснее, чтобы права страниц назначали только:

администратор Битрикс;

глобальный ADMIN;

глобальный OWNER.


А право делегирования позднее можно добавить отдельно как page.access.manage.

Что будет блокировать тест через интерфейс

Даже после исправления page.php пользователь с точечным canEdit пока не сможет открыть редактор, потому что в editor.php стоит:

if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}

Эта проверка пропускает только глобальные ADMIN и OWNER. Глобальный EDITOR и пользователи с точечным canEdit не проходят.

Следующим шагом нужно изменить вход в editor.php, чтобы:

OWNER, ADMIN, EDITOR могли открыть редактор;

пользователь с точечным page.edit тоже мог открыть редактор;

пользователь только с VIEWER/page.view не получал интерфейс редактирования.