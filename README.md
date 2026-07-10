site.php нужно доработать точечно. site.update, site.delete, site.setHome, управление ролями и оформлением пока оставляем привязанными к глобальным ролям.

1. Подключи сервисы точечных прав

Сразу после:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/SiteAppearanceService.php';

добавь:

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessRepository.php';
require_once $_SERVER['DOCUMENT_ROOT'] . '/local/sitebuilder/lib/PageAccessService.php';

2. Добавь общий метод определения доступа к сайту

После sb_site_handler_require_viewer() добавь:

if (!function_exists('sb_site_handler_get_access_context')) {
    function sb_site_handler_get_access_context(int $siteId): array
    {
        global $USER;

        if (
            $siteId <= 0
            || !is_object($USER)
            || !$USER->IsAuthorized()
        ) {
            return [
                'allowed' => false,
                'userId' => 0,
                'role' => '',
                'roleRank' => 0,
                'hasGlobalView' => false,
                'hasGlobalEdit' => false,
                'hasPageAccess' => false,
            ];
        }

        $userId = (int)$USER->GetID();

        if ($USER->IsAdmin()) {
            return [
                'allowed' => true,
                'userId' => $userId,
                'role' => 'OWNER',
                'roleRank' => 4,
                'hasGlobalView' => true,
                'hasGlobalEdit' => true,
                'hasPageAccess' => true,
            ];
        }

        $accessCode = PageAccessRepository::userAccessCode(
            $userId
        );

        /*
         * sb_get_role() учитывает как sitebuilder.access,
         * так и резервную роль группы Битрикс24.
         */
        $role = (string)sb_get_role(
            $siteId,
            $accessCode
        );

        $roleRank = sb_role_rank($role);

        $hasPageAccess = PageAccessService::hasAnyPageAccess(
            $siteId,
            $userId
        );

        return [
            'allowed' => $roleRank >= 1 || $hasPageAccess,
            'userId' => $userId,
            'role' => $role,
            'roleRank' => $roleRank,
            'hasGlobalView' => $roleRank >= 1,
            'hasGlobalEdit' => $roleRank >= 2,
            'hasPageAccess' => $hasPageAccess,
        ];
    }
}

3. Замени site.list

Полностью замени текущий блок:

if ($action === 'site.list') {
    // ...
}

на:

if ($action === 'site.list') {
    $sites = sb_read_sites();
    $allowedSites = [];

    foreach ($sites as $site) {
        $currentSiteId = (int)($site['id'] ?? 0);

        if ($currentSiteId <= 0) {
            continue;
        }

        $accessContext =
            sb_site_handler_get_access_context(
                $currentSiteId
            );

        /*
         * Сайт показывается, если пользователь:
         *
         * 1. Имеет глобальную роль VIEWER или выше.
         * 2. Либо имеет хотя бы одно точечное право страницы.
         */
        if (!$accessContext['allowed']) {
            continue;
        }

        $site['currentUserRole'] =
            $accessContext['role'];

        $site['currentUserRoleRank'] =
            $accessContext['roleRank'];

        $site['currentUserHasGlobalView'] =
            $accessContext['hasGlobalView'];

        $site['currentUserHasGlobalEdit'] =
            $accessContext['hasGlobalEdit'];

        $site['currentUserHasPageAccess'] =
            $accessContext['hasPageAccess'];

        $allowedSites[] = $site;
    }

    usort(
        $allowedSites,
        static function ($a, $b) {
            return
                (int)($a['id'] ?? 0)
                <=>
                (int)($b['id'] ?? 0);
        }
    );

    sb_json_ok([
        'sites' => $allowedSites,
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

Теперь сайт появится в списке даже у пользователя без глобальной роли, если ему выдали доступ хотя бы к одной странице.

4. Замени site.get

Полностью замени текущий блок:

if ($action === 'site.get') {
    // ...
}

на:

if ($action === 'site.get') {
    $siteId = (int)($_POST['siteId'] ?? 0);

    if ($siteId <= 0) {
        sb_json_error('SITE_ID_REQUIRED', 422);
    }

    /*
     * Сначала проверяем существование сайта,
     * затем его права.
     */
    $site = sb_find_site($siteId);

    if (!$site) {
        sb_json_error('SITE_NOT_FOUND', 404);
    }

    $accessContext =
        sb_site_handler_get_access_context($siteId);

    if (!$accessContext['allowed']) {
        sb_json_error(
            'SITE_OR_PAGE_ACCESS_DENIED',
            403,
            [
                'siteId' => $siteId,
            ]
        );
    }

    /*
     * Эти поля нужны клиентской части для понимания
     * уровня текущего пользователя.
     */
    $site['currentUserRole'] =
        $accessContext['role'];

    $site['currentUserRoleRank'] =
        $accessContext['roleRank'];

    $site['currentUserHasGlobalView'] =
        $accessContext['hasGlobalView'];

    $site['currentUserHasGlobalEdit'] =
        $accessContext['hasGlobalEdit'];

    $site['currentUserHasPageAccess'] =
        $accessContext['hasPageAccess'];

    sb_json_ok([
        'site' => $site,
        'access' => [
            'role' => $accessContext['role'],
            'roleRank' => $accessContext['roleRank'],
            'globalView' =>
                $accessContext['hasGlobalView'],
            'globalEdit' =>
                $accessContext['hasGlobalEdit'],
            'hasPageAccess' =>
                $accessContext['hasPageAccess'],
        ],
        'handler' => 'site',
        'file' => __FILE__,
    ]);
}

5. Очищай точечные права при удалении сайта

Внутри site.delete, после:

sb_write_access($access);

добавь:

/*
 * Удаляем точечные права страниц удалённого сайта.
 */
sb_db_execute("
    DELETE FROM sitebuilder.page_access
    WHERE site_id = :site_id
", [
    ':site_id' => $id,
]);

Иначе в sitebuilder.page_access будут оставаться осиротевшие записи.

Что получится

Пользователь	site.list	site.get	Редактор

Администратор Битрикс	Да	Да	Да
OWNER	Да	Да	Да
ADMIN	Да	Да	Да
EDITOR	Да	Да	Да
VIEWER	Да	Да	Нет
Только page.edit	Да	Да	Да
Только page.view	Да	Да	Нет
Без прав	Нет	Нет	Нет


40-access.js менять не требуется: у пользователя без OWNER запрос site.accessList получит отказ, а панели управления ролями будут автоматически скрыты.

Следующим нужно проверить block.php, потому что точечное page.edit должно обязательно применяться ко всем операциям с блоками страницы.