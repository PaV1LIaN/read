Идём дальше. Сейчас сделаем middleware для ролей проекта.

Важно: роли проекта — это не группы Битрикса.

Группы Битрикса:

group:1
groups:1,7,12

А роли проекта:

owner
admin
editor
viewer

Например в sitebuilder потом будет так:

$router->get('/sites/{id}/edit', [SiteController::class, 'edit'], ['auth', 'role:editor']);

То есть:

Пустить только того, у кого в этом проекте роль editor или выше.


---

1. Создай /local/mvc/Core/RoleResolverInterface.php

<?php

namespace Local\Mvc\Core;

/**
 * RoleResolverInterface
 *
 * Интерфейс для проверки ролей проекта.
 *
 * Простыми словами:
 * фреймворк спрашивает:
 * "У пользователя есть такая роль?"
 *
 * А конкретный проект отвечает:
 * "Да" или "Нет".
 */
interface RoleResolverInterface
{
    public function hasRole(string $role, ?int $userId = null): bool;

    public function roles(?int $userId = null): array;
}


---

2. Создай /local/mvc/Core/Role.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * Role
 *
 * Помощник для проверки проектных ролей.
 *
 * Сам фреймворк НЕ знает, откуда берутся роли.
 * Он берёт resolver из config.php проекта.
 */
class Role
{
    private static ?RoleResolverInterface $resolver = null;

    public static function has(string $role, ?int $userId = null): bool
    {
        return self::resolver()->hasRole($role, $userId);
    }

    public static function hasAny(array $roles, ?int $userId = null): bool
    {
        foreach ($roles as $role) {
            if (self::has((string)$role, $userId)) {
                return true;
            }
        }

        return false;
    }

    public static function all(?int $userId = null): array
    {
        return self::resolver()->roles($userId);
    }

    private static function resolver(): RoleResolverInterface
    {
        if (self::$resolver instanceof RoleResolverInterface) {
            return self::$resolver;
        }

        $class = (string)Config::get('roles.resolver', '');

        if ($class === '') {
            throw new RuntimeException('ROLE_RESOLVER_NOT_CONFIGURED');
        }

        if (!class_exists($class)) {
            throw new RuntimeException('ROLE_RESOLVER_CLASS_NOT_FOUND: ' . $class);
        }

        $resolver = new $class();

        if (!$resolver instanceof RoleResolverInterface) {
            throw new RuntimeException('ROLE_RESOLVER_MUST_IMPLEMENT_INTERFACE: ' . $class);
        }

        self::$resolver = $resolver;

        return self::$resolver;
    }
}


---

3. Обнови /local/mvc/Core/Middleware.php

Найди место после блоков group / groups и перед csrf.

Добавь туда:

/**
 * role:editor
 *
 * Проверяет одну проектную роль.
 */
if ($name === 'role') {
    if ($argument !== '' && Role::has($argument)) {
        return null;
    }

    return Response::json([
        'ok' => false,
        'error' => 'ROLE_REQUIRED',
        'details' => [
            'message' => 'Недостаточно прав. Требуется роль: ' . $argument,
            'required_role' => $argument,
            'user_roles' => Role::all(),
        ],
    ], 403);
}

/**
 * roles:owner,admin,editor
 *
 * Пускает, если есть хотя бы одна роль из списка.
 */
if ($name === 'roles') {
    $requiredRoles = self::parseRoleList($argument);

    if (Role::hasAny($requiredRoles)) {
        return null;
    }

    return Response::json([
        'ok' => false,
        'error' => 'ROLES_REQUIRED',
        'details' => [
            'message' => 'Недостаточно прав. Требуется одна из ролей.',
            'required_roles' => $requiredRoles,
            'user_roles' => Role::all(),
        ],
    ], 403);
}

Теперь в конец класса Middleware, рядом с parseGroupList(), добавь метод:

private static function parseRoleList(string $argument): array
{
    $items = explode(',', $argument);
    $roles = [];

    foreach ($items as $item) {
        $role = trim((string)$item);

        if ($role !== '') {
            $roles[] = $role;
        }
    }

    return array_values(array_unique($roles));
}


---

4. Создай role resolver для demo-проекта

Файл:

/local/mvc_demo/Services/DemoRoleResolver.php

<?php

namespace Local\MvcDemo\Services;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\RoleResolverInterface;

/**
 * DemoRoleResolver
 *
 * Тестовая проверка ролей для mvc_demo.
 *
 * В реальном проекте роли можно брать:
 * - из таблицы доступа
 * - из HL-блока
 * - из PostgreSQL
 * - из групп Битрикса
 */
class DemoRoleResolver implements RoleResolverInterface
{
    public function hasRole(string $role, ?int $userId = null): bool
    {
        return in_array($role, $this->roles($userId), true);
    }

    public function roles(?int $userId = null): array
    {
        if (!Auth::check()) {
            return [];
        }

        /**
         * Для demo:
         *
         * Админ Битрикса получает все роли.
         */
        if (Auth::isAdmin()) {
            return [
                'owner',
                'admin',
                'editor',
                'viewer',
            ];
        }

        /**
         * Обычный авторизованный пользователь получает viewer.
         */
        return [
            'viewer',
        ];
    }
}


---

5. Подключи resolver в /local/mvc_demo/config.php

Добавь блок:

'roles' => [
    'resolver' => \Local\MvcDemo\Services\DemoRoleResolver::class,
],

Полный файл будет примерно такой:

<?php

return [
    'app' => [
        'name' => 'MVC Demo',
        'description' => 'Тестовый проект на общем MVC-фреймворке',
    ],

    'debug' => true,

    'log' => [
        'file' => __DIR__ . '/logs/app.log',
    ],

    'roles' => [
        'resolver' => \Local\MvcDemo\Services\DemoRoleResolver::class,
    ],
];


---

6. Добавь тестовые методы в HomeController

Файл:

/local/mvc_demo/Controllers/HomeController.php

Добавь внутрь класса:

public function roleViewerTest(): Response
{
    return $this->success([
        'message' => 'Доступ по роли viewer разрешён.',
        'roles' => \Local\Mvc\Core\Role::all(),
    ]);
}

public function roleEditorTest(): Response
{
    return $this->success([
        'message' => 'Доступ по роли editor разрешён.',
        'roles' => \Local\Mvc\Core\Role::all(),
    ]);
}


---

7. Добавь маршруты в /local/mvc_demo/routes.php

$router->get('/role-viewer-test', [HomeController::class, 'roleViewerTest'], ['auth', 'role:viewer']);

$router->get('/role-editor-test', [HomeController::class, 'roleEditorTest'], ['auth', 'role:editor']);


---

8. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/role-viewer-test

Если пользователь авторизован, должно работать.

Потом:

https://bitrix24-stage.gaz.ru/local/mvc_demo/role-editor-test

Если ты админ Битрикса — тоже будет работать.

Если обычный пользователь — будет ошибка:

{
  "ok": false,
  "error": "ROLE_REQUIRED",
  "details": {
    "message": "Недостаточно прав. Требуется роль: editor",
    "required_role": "editor",
    "user_roles": ["viewer"]
  }
}


---

Что мы сделали

Теперь фреймворк умеет проверять не только группы Битрикса:

['group:1']

но и проектные роли:

['role:editor']
['roles:owner,admin']

Главная мысль:

Фреймворк не знает, где хранятся роли.
Проект сам говорит фреймворку, как роли проверять.

Для будущего sitebuilder это будет идеально:

$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/sites', [SiteController::class, 'index'], ['role:viewer']);
    $router->get('/sites/create', [SiteController::class, 'create'], ['role:editor']);
    $router->post('/sites/store', [SiteController::class, 'store'], ['role:editor', 'csrf']);
    $router->post('/sites/delete', [SiteController::class, 'delete'], ['role:owner', 'csrf']);
});

Дальше можно сделать иерархию ролей, чтобы owner автоматически был выше admin, admin выше editor, а editor выше viewer.