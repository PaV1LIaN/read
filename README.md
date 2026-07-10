40-access.js менять не нужно. Он уже корректно работает для ограниченных пользователей:

try {
    var res = await api('site.accessList', {
        siteId: siteId
    });

    // ...
} catch (e) {
    state.accessItems = [];
    setManagementPanelsVisible(false);
}

Если пользователь не OWNER, сервер отклонит site.accessList, ошибка будет перехвачена, а панели прав просто скроются.

Также setManagementPanelsVisible(false) скрывает:

управление группой Битрикс24;

глобальные роли;

технический ответ API;

кнопку удаления сайта.


Что изменить в editor.php

Добавь в $libFiles два файла:

__DIR__ . '/lib/PageAccessRepository.php',
__DIR__ . '/lib/PageAccessService.php',

Должно получиться:

$libFiles = [
    __DIR__ . '/lib/db.php',
    __DIR__ . '/lib/json.php',
    __DIR__ . '/lib/storage_db.php',
    __DIR__ . '/lib/response.php',
    __DIR__ . '/lib/helpers.php',
    __DIR__ . '/lib/access.php',
    __DIR__ . '/lib/PageAccessRepository.php',
    __DIR__ . '/lib/PageAccessService.php',
];

После проверки $siteId замени:

if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}

на:

$currentUserId = (int)$USER->GetID();

$canOpenEditor = false;

if ($USER->IsAdmin()) {
    $canOpenEditor = true;
}

/*
 * Глобальные роли:
 * EDITOR, ADMIN и OWNER.
 *
 * Используем access.php, потому что он также учитывает
 * резервные роли группы Битрикс24.
 */
if (!$canOpenEditor) {
    $globalRole = sb_get_role($siteId);
    $globalRoleRank = sb_role_rank($globalRole);

    if ($globalRoleRank >= 2) {
        $canOpenEditor = true;
    }
}

/*
 * Пользователь без глобальной роли может открыть редактор,
 * если у него есть canEdit хотя бы на одну страницу.
 */
if (!$canOpenEditor && $currentUserId > 0) {
    $accessCode = PageAccessRepository::userAccessCode(
        $currentUserId
    );

    $pageIds = PageAccessRepository::getPageIdsWithAccess(
        $siteId,
        $accessCode
    );

    foreach ($pageIds as $availablePageId) {
        if (
            PageAccessService::canEditPage(
                $siteId,
                (int)$availablePageId,
                $currentUserId
            )
        ) {
            $canOpenEditor = true;
            break;
        }
    }
}

if (!$canOpenEditor) {
    http_response_code(403);

    ?>
    <!doctype html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>Доступ запрещён</title>
        <?php $APPLICATION->ShowHead(); ?>

        <link
            rel="stylesheet"
            href="<?= htmlspecialchars(
                $basePath,
                ENT_QUOTES | ENT_SUBSTITUTE,
                'UTF-8'
            ) ?>/assets/admin/admin.css"
        >
    </head>

    <body class="sb-admin-body">
    <div class="sb-page">
        <h1 class="sb-title">Доступ к редактору запрещён</h1>

        <p class="sb-subtitle">
            Для открытия редактора требуется глобальная роль
            EDITOR, ADMIN или OWNER либо право редактирования
            хотя бы одной страницы.
        </p>

        <p>
            <a
                class="sb-back-link"
                href="<?= htmlspecialchars(
                    $basePath,
                    ENT_QUOTES | ENT_SUBSTITUTE,
                    'UTF-8'
                ) ?>/index.php"
            >
                Вернуться к списку сайтов
            </a>
        </p>
    </div>
    </body>
    </html>
    <?php

    exit;
}

После этого вход будет таким:

Доступ	Открытие редактора

Администратор Битрикс	Да
OWNER	Да
ADMIN	Да
EDITOR	Да
VIEWER	Нет
Только page.view	Нет
Хотя бы одно page.edit	Да
Нет прав	Нет


Что ещё блокирует точечного пользователя

После открытия страницы JavaScript запрашивает информацию о сайте через site.get. По текущей архитектуре этот API, вероятнее всего, проверяет глобальный VIEWER.

Поэтому пользователь только с page.edit сможет пройти PHP-проверку editor.php, но может получить отказ на этапе загрузки site.get.

Следующим нужно проверить:

/local/sitebuilder/api/handlers/site.php

Там надо разрешить site.get пользователю, если у него есть хотя бы одно точечное право страницы, не открывая ему управление настройками сайта.