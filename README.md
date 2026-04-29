Нашёл. Нужный файл:

```text
/local/sitebuilder/components/disk/lib/DiskPermissionService.php
```

И важно: серверные проверки уже стоят в actions:

```text
actions/upload.php          → canUpload
actions/create_folder.php   → canCreateFolder
actions/rename.php          → canRename
actions/delete.php          → canDelete
actions/download.php        → canDownload
actions/save_settings.php   → canEditSettings
```

То есть достаточно заменить `DiskPermissionService.php`, потому что все действия уже проходят через:

```php
DiskPermissionService::resolve(...)
DiskValidator::assertCan(...)
```

Замени файл:

```text
/local/sitebuilder/components/disk/lib/DiskPermissionService.php
```

на этот:

```php
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

            /*
             * Эти права не зависят от настроек блока.
             * Их определяет только роль пользователя на сайте.
             */
            'canManageAccess' => $rolePermissions['canManageAccess'],
            'canEditSettings' => $rolePermissions['canEditSettings'],

            /*
             * Для отладки и интерфейса.
             */
            'role' => $rolePermissions['role'],
        ];
    }

    protected static function resolveRolePermissions(DiskContext $context): array
    {
        if (DiskCurrentUser::isAdmin()) {
            return self::permissionsForRole('bitrix_admin');
        }

        $role = SiteAccessRepository::getUserRole($context->siteId, $context->currentUserId);

        return self::permissionsForRole((string)$role);
    }

    protected static function permissionsForRole(string $role): array
    {
        $role = trim($role);

        /*
         * OWNER / ADMIN сайта.
         * Полный доступ к диску в рамках сайта.
         */
        if ($role === 'site_admin' || $role === 'bitrix_admin') {
            return [
                'role' => $role,
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

        /*
         * EDITOR сайта.
         * Может работать с файлами, но не может удалять
         * и не может менять настройки компонента.
         */
        if ($role === 'site_editor') {
            return [
                'role' => $role,
                'canView' => true,
                'canUpload' => true,
                'canCreateFolder' => true,
                'canRename' => true,
                'canDelete' => false,
                'canDownload' => true,
                'canManageAccess' => false,
                'canEditSettings' => false,
            ];
        }

        /*
         * Старый/промежуточный вариант роли, если где-то остался USER/MEMBER.
         * Оставляем как ограниченного редактора без удаления и без настроек.
         */
        if ($role === 'site_user') {
            return [
                'role' => $role,
                'canView' => true,
                'canUpload' => true,
                'canCreateFolder' => true,
                'canRename' => false,
                'canDelete' => false,
                'canDownload' => true,
                'canManageAccess' => false,
                'canEditSettings' => false,
            ];
        }

        /*
         * VIEWER сайта.
         * Только просмотр и скачивание.
         */
        if ($role === 'site_viewer') {
            return [
                'role' => $role,
                'canView' => true,
                'canUpload' => false,
                'canCreateFolder' => false,
                'canRename' => false,
                'canDelete' => false,
                'canDownload' => true,
                'canManageAccess' => false,
                'canEditSettings' => false,
            ];
        }

        /*
         * Нет роли на сайте — нет доступа к диску.
         */
        return [
            'role' => '',
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
```

Что изменится:

```text
VIEWER
- видит файлы
- скачивает

EDITOR
- видит файлы
- скачивает
- загружает
- создаёт папки
- переименовывает
- НЕ удаляет
- НЕ меняет настройки диска

ADMIN / OWNER / админ Битрикс24
- полный доступ
```

После замены проверь под пользователем с ролью `EDITOR`: кнопка удаления в диске должна пропасть или стать недоступной, а прямой запрос на удаление через API должен вернуть ошибку доступа.
