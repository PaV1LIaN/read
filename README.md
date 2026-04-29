Дальше делаем 3 правки: закрываем `editor.php` от роли `EDITOR`, убираем кнопку **Редактор** на главной у `EDITOR`, и меняем права диска.

## 1. Закрыть `editor.php` от EDITOR

Файл:

```text
/local/sitebuilder/editor.php
```

Найди вверху:

```php
$basePath = rtrim(str_replace($_SERVER['DOCUMENT_ROOT'], '', __DIR__), '/');
$siteId = (int)($_GET['siteId'] ?? 0);
```

Сразу после этого добавь:

```php
require_once __DIR__ . '/lib/json.php';
require_once __DIR__ . '/lib/response.php';
require_once __DIR__ . '/lib/access.php';
require_once __DIR__ . '/lib/helpers.php';
```

Дальше найди конец блока:

```php
if ($siteId <= 0) {
    ?>
    ...
    <?php
    exit;
}
?>
```

И **перед** закрывающим `?>` добавь:

```php
if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}
```

Должно получиться примерно так:

```php
if ($siteId <= 0) {
    ?>
    <!doctype html>
    <html lang="ru">
    ...
    </html>
    <?php
    exit;
}

if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}
?>
```

Теперь `EDITOR` при прямом открытии `editor.php?siteId=...` получит `ACCESS_DENIED`.

---

## 2. Убрать кнопку «Редактор» у EDITOR на главной

Файл:

```text
/local/sitebuilder/index.php
```

Найди все проверки:

```js
>= 2
```

которые относятся к показу кнопки **Редактор**.

В твоём файле это обычно два места:

```js
if (userRoleRank >= 2) {
```

и:

```js
(currentUserRoleRank >= 2
    ? '<a class="sb-btn sb-btn-primary sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + siteId + '">Редактор</a>'
    : '')
```

Замени их на:

```js
>= 3
```

То есть должно стать:

```js
if (userRoleRank >= 3) {
```

и:

```js
(currentUserRoleRank >= 3
    ? '<a class="sb-btn sb-btn-primary sb-btn-small" href="' + BASE_PATH + '/editor.php?siteId=' + siteId + '">Редактор</a>'
    : '')
```

Теперь кнопка **Редактор** будет только у `ADMIN / OWNER / администратора Битрикс24`.

---

## 3. Заменить права диска

Файл:

```text
/local/sitebuilder/components/disk/lib/DiskPermissionService.php
```

Замени полностью на этот код:

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

            'canManageAccess' => $rolePermissions['canManageAccess'],
            'canEditSettings' => $rolePermissions['canEditSettings'],

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
         * EDITOR теперь НЕ редактирует сайт.
         * Он работает только с файлами диска:
         * загрузка, скачивание, удаление.
         */
        if ($role === 'site_editor') {
            return [
                'role' => $role,
                'canView' => true,
                'canUpload' => true,
                'canCreateFolder' => false,
                'canRename' => false,
                'canDelete' => true,
                'canDownload' => true,
                'canManageAccess' => false,
                'canEditSettings' => false,
            ];
        }

        if ($role === 'site_user') {
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

После этого матрица станет такой:

```text
VIEWER:
- публичная часть
- просмотр диска
- скачивание

EDITOR:
- публичная часть
- просмотр диска
- скачивание
- загрузка файлов
- удаление файлов/объектов диска
- без доступа к editor.php

ADMIN / OWNER:
- editor.php
- страницы
- блоки
- меню
- layout
- полный диск
```

Важно: сейчас `canDelete` в диске разрешает удаление объекта диска. Если нужно строго разрешить `EDITOR` удалять только файлы, но не папки, следующим шагом отдельно поправим `components/disk/actions/delete.php`.
