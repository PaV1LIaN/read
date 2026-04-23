Отлично. Тогда делаем единые права так, чтобы:

sitebuilder и disk смотрели на один и тот же источник прав

права брались из таблицы sitebuilder.access

disk больше не зависел от старой таблицы sitebuilder_site_user_access



---

Что меняем

Сейчас у тебя в disk права живут через:

/local/sitebuilder/components/disk/lib/SiteAccessRepository.php
/local/sitebuilder/components/disk/lib/DiskPermissionService.php

Нужно сделать так, чтобы disk читал те же access rows, что и сам sitebuilder.


---

Логика общей модели прав

Берем роли из sitebuilder.access и маппим их в права disk.

Маппинг

OWNER / ADMIN   -> site_admin
EDITOR          -> site_editor
USER / MEMBER   -> site_user
VIEWER / READER -> site_viewer

Access code

Для обычного пользователя Bitrix используем:

U<ID>

Например:

пользователь 1 → U1

пользователь 25 → U25


Дополнительно можно поддержать группы:

G<ID группы>


И отдельный супер-код:

AU для админа системы, если ты его используешь в sitebuilder



---

1. Замени файл /local/sitebuilder/components/disk/lib/SiteAccessRepository.php

<?php

class SiteAccessRepository
{
    public static function getUserRole(int $siteId, int $userId): ?string
    {
        if ($siteId <= 0 || $userId <= 0) {
            return null;
        }

        $accessCodes = self::buildAccessCodes($userId);
        if (empty($accessCodes)) {
            return null;
        }

        $placeholders = [];
        $params = [
            ':site_id' => $siteId,
        ];

        foreach ($accessCodes as $index => $code) {
            $key = ':code_' . $index;
            $placeholders[] = $key;
            $params[$key] = $code;
        }

        $sql = "
            SELECT access_code, role
            FROM sitebuilder.access
            WHERE site_id = :site_id
              AND access_code IN (" . implode(', ', $placeholders) . ")
        ";

        $rows = DiskDb::fetchAll($sql, $params);
        if (empty($rows)) {
            return null;
        }

        $bestRole = null;
        $bestRank = -1;

        foreach ($rows as $row) {
            $roleCode = strtoupper(trim((string)($row['role'] ?? '')));
            $mappedRole = self::mapSitebuilderRoleToDiskRole($roleCode);

            if ($mappedRole === null) {
                continue;
            }

            $rank = self::diskRoleRank($mappedRole);
            if ($rank > $bestRank) {
                $bestRank = $rank;
                $bestRole = $mappedRole;
            }
        }

        return $bestRole;
    }

    public static function setUserRole(int $siteId, int $userId, string $roleCode): bool
    {
        if ($siteId <= 0 || $userId <= 0) {
            throw new RuntimeException('INVALID_SITE_OR_USER');
        }

        $accessCode = 'U' . $userId;
        $sitebuilderRole = self::mapDiskRoleToSitebuilderRole($roleCode);

        if ($sitebuilderRole === null) {
            throw new RuntimeException('INVALID_ROLE_CODE');
        }

        $sql = "
            INSERT INTO sitebuilder.access (
                site_id,
                access_code,
                role,
                created_at,
                updated_at
            ) VALUES (
                :site_id,
                :access_code,
                :role,
                CURRENT_TIMESTAMP,
                CURRENT_TIMESTAMP
            )
            ON CONFLICT (site_id, access_code)
            DO UPDATE SET
                role = EXCLUDED.role,
                updated_at = CURRENT_TIMESTAMP
        ";

        return DiskDb::execute($sql, [
            ':site_id' => $siteId,
            ':access_code' => $accessCode,
            ':role' => $sitebuilderRole,
        ]);
    }

    public static function hasAnyAccess(int $siteId, int $userId): bool
    {
        return self::getUserRole($siteId, $userId) !== null;
    }

    protected static function buildAccessCodes(int $userId): array
    {
        $codes = ['U' . $userId];

        if (DiskCurrentUser::isAdmin()) {
            $codes[] = 'AU';
            $codes[] = 'ADMIN';
        }

        foreach (DiskCurrentUser::getGroupIds() as $groupId) {
            if ($groupId > 0) {
                $codes[] = 'G' . (int)$groupId;
            }
        }

        return array_values(array_unique($codes));
    }

    protected static function mapSitebuilderRoleToDiskRole(string $roleCode): ?string
    {
        switch ($roleCode) {
            case 'OWNER':
            case 'ADMIN':
                return 'site_admin';

            case 'EDITOR':
                return 'site_editor';

            case 'USER':
            case 'MEMBER':
                return 'site_user';

            case 'VIEWER':
            case 'READER':
                return 'site_viewer';

            default:
                return null;
        }
    }

    protected static function mapDiskRoleToSitebuilderRole(string $roleCode): ?string
    {
        switch ($roleCode) {
            case 'site_admin':
                return 'OWNER';

            case 'site_editor':
                return 'EDITOR';

            case 'site_user':
                return 'USER';

            case 'site_viewer':
                return 'VIEWER';

            default:
                return null;
        }
    }

    protected static function diskRoleRank(string $roleCode): int
    {
        switch ($roleCode) {
            case 'site_admin':
                return 4;

            case 'site_editor':
                return 3;

            case 'site_user':
                return 2;

            case 'site_viewer':
                return 1;

            default:
                return 0;
        }
    }
}


---

2. DiskPermissionService.php можно оставить почти как есть

Если у тебя уже стоит вот такая логика:

DiskCurrentUser::isAdmin() → полный доступ

дальше SiteAccessRepository::getUserRole(...)


то после замены SiteAccessRepository.php права уже станут общими.

Но я все равно пришлю готовую финальную версию, чтобы у тебя не было расхождений.

/local/sitebuilder/components/disk/lib/DiskPermissionService.php

<?php

class DiskPermissionService
{
    public static function resolve(DiskContext $context, array $settings, ?int $rootFolderId = null): array
    {
        $rolePermissions = self::resolveRolePermissions($context);
        $blockRestrictions = self::resolveBlockRestrictions($settings);

        return [
            'canView' => $rolePermissions['canView'] && $blockRestrictions['canView'],
            'canUpload' => $rolePermissions['canUpload'] && $blockRestrictions['canUpload'],
            'canCreateFolder' => $rolePermissions['canCreateFolder'] && $blockRestrictions['canCreateFolder'],
            'canRename' => $rolePermissions['canRename'] && $blockRestrictions['canRename'],
            'canDelete' => $rolePermissions['canDelete'] && $blockRestrictions['canDelete'],
            'canDownload' => $rolePermissions['canDownload'] && $blockRestrictions['canDownload'],
            'canManageAccess' => $rolePermissions['canManageAccess'],
            'canEditSettings' => $rolePermissions['canEditSettings'],
        ];
    }

    protected static function resolveRolePermissions(DiskContext $context): array
    {
        if (DiskCurrentUser::isAdmin()) {
            return [
                'canView' => true,
                'canUpload' => true,
                'canCreateFolder' => true,
                'canRename' => true,
                'canDelete' => true,
                'canDownload' => true,
                'canManageAccess' => true,
                'canEditSettings' => true,
            ];
        }

        $role = SiteAccessRepository::getUserRole($context->siteId, $context->currentUserId);

        switch ($role) {
            case 'site_admin':
                return [
                    'canView' => true,
                    'canUpload' => true,
                    'canCreateFolder' => true,
                    'canRename' => true,
                    'canDelete' => true,
                    'canDownload' => true,
                    'canManageAccess' => true,
                    'canEditSettings' => true,
                ];

            case 'site_editor':
                return [
                    'canView' => true,
                    'canUpload' => true,
                    'canCreateFolder' => true,
                    'canRename' => true,
                    'canDelete' => true,
                    'canDownload' => true,
                    'canManageAccess' => false,
                    'canEditSettings' => true,
                ];

            case 'site_user':
                return [
                    'canView' => true,
                    'canUpload' => true,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => true,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ];

            case 'site_viewer':
                return [
                    'canView' => true,
                    'canUpload' => false,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => true,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ];

            default:
                return [
                    'canView' => false,
                    'canUpload' => false,
                    'canCreateFolder' => false,
                    'canRename' => false,
                    'canDelete' => false,
                    'canDownload' => false,
                    'canManageAccess' => false,
                    'canEditSettings' => false,
                ];
        }
    }

    protected static function resolveBlockRestrictions(array $settings): array
    {
        return [
            'canView' => true,
            'canUpload' => !empty($settings['allowUpload']),
            'canCreateFolder' => !empty($settings['allowCreateFolder']),
            'canRename' => !empty($settings['allowRename']),
            'canDelete' => !empty($settings['allowDelete']),
            'canDownload' => !empty($settings['allowDownload']),
        ];
    }
}


---

3. Что это даст

После замены этих файлов:

editor.php, sitebuilder, access.php и disk будут смотреть на одни и те же access rows

если пользователь имеет в sitebuilder.access роль OWNER, то disk увидит site_admin

если роль EDITOR, то disk даст права редактора

если VIEWER, то в disk будет только просмотр



---

4. Что проверить после правки

Проверь на одном сайте:

Пользователь с OWNER

Должен мочь:

открывать диск

загружать

создавать папки

удалять

переименовывать

открывать настройки блока


Пользователь с EDITOR

Должен мочь:

открывать диск

загружать

создавать папки

редактировать

открывать настройки блока

но без управления доступами


Пользователь с VIEWER

Должен мочь:

только смотреть

скачивать

но не загружать и не удалять



---

5. Что еще желательно сделать

Если ты больше не используешь старую таблицу:

sitebuilder_site_user_access

то ее лучше:

либо оставить как legacy и больше не трогать,

либо потом удалить из проекта окончательно.


Но сначала убедись, что весь disk уже работает от sitebuilder.access.


---

6. Следующий логичный шаг

После объединения прав я бы следующим шагом сделал:

единый repository/service слой для sitebuilder и disk, чтобы:

не было двух параллельных моделей SiteRepository

DiskRootResolver и sitebuilder использовали один источник данных по сайту

disk стал полноценным блоком конструктора, а не отдельным островом


Если хочешь, следующим сообщением я дам тебе план и код, как связать sitebuilder.site и disk по rootFolder/diskFolderId` без дублирования логики.