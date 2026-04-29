Да, идём. Я посмотрел структуру проекта. Сейчас нужно сделать 5 правок.

## 1. В `access.php` добавь новую функцию

Файл:

```text
/local/sitebuilder/lib/access.php
```

Найди функцию:

```php
function sb_require_site_role(int $siteId, int $minRank): void
{
    $role = sb_get_role($siteId, sb_user_access_code());

    if (sb_role_rank($role) < $minRank) {
        sb_json_error('ACCESS_DENIED', 403, [
            'siteId' => $siteId,
            'requiredRank' => $minRank,
            'actualRole' => $role,
        ]);
    }
}
```

Замени её на:

```php
function sb_require_site_role(int $siteId, int $minRank): void
{
    global $USER;

    if ($USER && $USER->IsAdmin()) {
        return;
    }

    $role = sb_get_role($siteId, sb_user_access_code());

    if (sb_role_rank($role) < $minRank) {
        sb_json_error('ACCESS_DENIED', 403, [
            'siteId' => $siteId,
            'requiredRank' => $minRank,
            'actualRole' => $role,
        ]);
    }
}
```

Теперь администратор Битрикс24 проходит все проверки.

---

## 2. В этот же `access.php` добавь новую функцию

После функции:

```php
function sb_require_admin(int $siteId): void
{
    sb_require_site_role($siteId, 3);
}
```

добавь:

```php
if (!function_exists('sb_require_content_manager')) {
    function sb_require_content_manager(int $siteId): void
    {
        /*
         * Контент сайта: страницы, блоки, меню, layout, шаблоны.
         * Доступ только ADMIN / OWNER / админ Битрикс24.
         *
         * EDITOR сюда НЕ проходит, потому что EDITOR теперь только для работы с файлами диска.
         */
        sb_require_site_role($siteId, 3);
    }
}
```

---

## 3. Закрываем страницы, блоки, меню и layout от `EDITOR`

Нужно заменить проверки:

```php
sb_require_editor($siteId);
```

на:

```php
sb_require_content_manager($siteId);
```

в этих файлах:

```text
/local/sitebuilder/api/handlers/page.php
/local/sitebuilder/api/handlers/block.php
/local/sitebuilder/api/handlers/menu.php
/local/sitebuilder/api/handlers/layout.php
/local/sitebuilder/api/handlers/template.php
```

Можно сделать командой на сервере:

```bash
perl -pi -e 's/sb_require_editor\(\$siteId\);/sb_require_content_manager(\$siteId);/g' \
/srv/bx/docroot/local/sitebuilder/api/handlers/page.php \
/srv/bx/docroot/local/sitebuilder/api/handlers/block.php \
/srv/bx/docroot/local/sitebuilder/api/handlers/menu.php \
/srv/bx/docroot/local/sitebuilder/api/handlers/layout.php \
/srv/bx/docroot/local/sitebuilder/api/handlers/template.php
```

Файл:

```text
/local/sitebuilder/api/handlers/file.php
```

пока не трогай. Это не дисковый компонент, там может быть загрузка логотипа/служебных файлов, отдельно проверим позже.

---

## 4. В `site.php` усиливаем `site.update` и `site.setHome`

Файл:

```text
/local/sitebuilder/api/handlers/site.php
```

Найди функцию:

```php
function sb_site_handler_require_editor(int $siteId): void
{
    sb_site_handler_require_role($siteId, 2);
}
```

Замени на:

```php
function sb_site_handler_require_editor(int $siteId): void
{
    /*
     * В старом коде эта функция называлась editor,
     * но теперь редактировать структуру сайта может только ADMIN / OWNER.
     */
    sb_site_handler_require_role($siteId, 3);
}
```

Так `site.update` и `site.setHome` больше не будут доступны роли `EDITOR`.

---

## 5. Закрываем сам `editor.php` от роли `EDITOR`

Файл:

```text
/local/sitebuilder/editor.php
```

После строки:

```php
$siteId = (int)($_GET['siteId'] ?? 0);
```

добавь:

```php
require_once __DIR__ . '/lib/db.php';
require_once __DIR__ . '/lib/json.php';
require_once __DIR__ . '/lib/response.php';
require_once __DIR__ . '/lib/access.php';
require_once __DIR__ . '/lib/helpers.php';
```

А после проверки:

```php
if ($siteId <= 0) {
```

и всего блока с `exit;`, перед `?>` добавь:

```php
if (!$USER->IsAdmin()) {
    sb_require_content_manager($siteId);
}
```

В итоге `EDITOR` при прямом открытии:

```text
/local/sitebuilder/editor.php?siteId=...
```

получит `ACCESS_DENIED`.

---

## 6. На главной убираем кнопку «Редактор» у `EDITOR`

Файл:

```text
/local/sitebuilder/index.php
```

Найди все места:

```js
currentUserRoleRank >= 2
```

и замени на:

```js
currentUserRoleRank >= 3
```

В твоём `index.php` таких мест обычно два:

```js
if (userRoleRank >= 2) {
```

и:

```js
(currentUserRoleRank >= 2 ? ...
```

Их тоже меняем на `>= 3`.

То есть кнопка **Редактор** теперь будет только у:

```text
ADMIN / OWNER / админ Битрикс24
```

---

## 7. Обновляем права диска для `EDITOR`

Файл:

```text
/local/sitebuilder/components/disk/lib/DiskPermissionService.php
```

В блоке роли:

```php
case 'site_editor':
```

замени права на:

```php
case 'site_editor':
    return [
        'canView' => true,
        'canUpload' => true,
        'canCreateFolder' => false,
        'canRename' => false,
        'canDelete' => true,
        'canDownload' => true,
        'canManageAccess' => false,
        'canEditSettings' => false,
    ];
```

Теперь `EDITOR`:

```text
может смотреть диск
может скачивать
может загружать файлы
может удалять файлы/объекты диска
не может менять страницы/блоки/меню/layout
не может открывать editor.php
```

После этих правок проверь под пользователем с ролью `EDITOR`:

```text
1. Главная — сайт виден.
2. Кнопки «Редактор» нет.
3. Публичная часть открывается.
4. Диск виден.
5. Загрузка файла работает.
6. Удаление файла работает.
7. Прямой переход на editor.php возвращает ACCESS_DENIED.
```
