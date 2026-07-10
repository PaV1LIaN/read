Да, этот репозиторий реализует нужную базовую модель прав на страницы:

прямой canView;

прямой canEdit;

canEdit автоматически включает просмотр;

includeChildren=true наследует право на всех потомков;

права складываются с глобальными ролями;

явных запретов пока нет.


Фактическое поведение

Доступ пользователя	Результат

Глобальный VIEWER	Просмотр всех страниц
Глобальный EDITOR	Просмотр и изменение всех страниц
Глобальный ADMIN	Изменение всех страниц и управление
Глобальный OWNER	Полный доступ
Нет глобальной роли	Только страницы из page_access
Право родителя с includeChildren=true	Действует на всех потомков


Это соответствует принятой модели.

Но перед тестированием нужно закрыть две реальные ошибки.

1. Проверять принадлежность страницы сайту

Сейчас можно передать:

siteId = сайт, где пользователь ADMIN
pageId = страница другого сайта

И save() не проверит, что эта страница принадлежит указанному сайту.

Добавь в PageAccessRepository публичный метод:

public static function pageBelongsToSite(int $siteId, int $pageId): bool
{
    if ($siteId <= 0 || $pageId <= 0) {
        return false;
    }

    $pdo = sb_db();

    $stmt = $pdo->prepare("
        SELECT 1
        FROM sitebuilder.page
        WHERE site_id = :site_id
          AND id = :page_id
        LIMIT 1
    ");

    $stmt->execute([
        ':site_id' => $siteId,
        ':page_id' => $pageId,
    ]);

    return (bool)$stmt->fetchColumn();
}

Также добавь:

public static function requirePageInSite(int $siteId, int $pageId): void
{
    if (!self::pageBelongsToSite($siteId, $pageId)) {
        throw new RuntimeException('PAGE_NOT_IN_SITE');
    }
}

В listByPage() после проверки ID:

self::requirePageInSite($siteId, $pageId);

В save() после проверок ID:

self::requirePageInSite($siteId, $pageId);

В delete() — если переданы оба значения:

if ($siteId > 0 && $pageId > 0) {
    self::requirePageInSite($siteId, $pageId);
}

2. Проверять страницу до определения возможности управления

В page_access.php функция сейчас сначала может разрешить доступ по глобальной роли сайта, не удостоверившись, что страница относится к этому сайту.

Замени её на:

if (!function_exists('sb_page_access_can_manage')) {
    function sb_page_access_can_manage(
        int $siteId,
        int $pageId,
        int $userId
    ): bool {
        global $USER;

        if (
            $siteId <= 0
            || $pageId <= 0
            || $userId <= 0
            || !PageAccessRepository::pageBelongsToSite($siteId, $pageId)
        ) {
            return false;
        }

        if (
            is_object($USER)
            && method_exists($USER, 'IsAdmin')
            && $USER->IsAdmin()
        ) {
            return true;
        }

        if (
            PageAccessService::hasGlobalSiteAccess(
                $siteId,
                $userId,
                'admin'
            )
        ) {
            return true;
        }

        return PageAccessService::canEditPage(
            $siteId,
            $pageId,
            $userId
        );
    }
}

Теперь невозможно использовать права одного сайта для управления правами страницы другого сайта.

3. Исправить результат удаления

Сейчас delete() всегда возвращает true, даже если запись не найдена.

Замени:

$stmt->execute($params);

return true;

на:

$stmt->execute($params);

return $stmt->rowCount() > 0;

А в page_access.php:

$deleted = PageAccessRepository::delete(
    $id,
    $siteId,
    $pageId
);

if (!$deleted) {
    throw new RuntimeException('PAGE_ACCESS_NOT_FOUND');
}

4. Ограничить формат accessCode

Сейчас разрешена любая буква:

/^[A-Z]\d+$/

Например, пройдут X12 или Z900.

Если пока права выдаются только пользователям, замени на:

if (!preg_match('/^U[1-9]\d*$/', $accessCode)) {
    throw new RuntimeException('INVALID_ACCESS_CODE');
}

Если планируется выдача группам, можно сразу разрешить два типа:

if (!preg_match('/^(U|G)[1-9]\d*$/', $accessCode)) {
    throw new RuntimeException('INVALID_ACCESS_CODE');
}

5. Проверить уникальный индекс PostgreSQL

Этот запрос:

ON CONFLICT (site_id, page_id, access_code)

работает только при наличии уникального ограничения.

В таблице должно быть:

CREATE UNIQUE INDEX IF NOT EXISTS uq_page_access_site_page_code
ON sitebuilder.page_access (
    site_id,
    page_id,
    access_code
);

Что пока не реализовано

В этой таблице присутствуют только:

can_view
can_edit
include_children

Прав для Диска пока нет:

disk.view
disk.edit
disk.download
disk.upload
disk.delete

Сначала нужно полностью проверить применение canView/canEdit в страницах и блоках. Следующим нужен файл:

/local/sitebuilder/api/handlers/page.php

В нём проверим, действительно ли пользователь без глобальной роли получает только разрешённые страницы и не может изменить чужую страницу прямым API-запросом.