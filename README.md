Идём дальше. Делаем иерархию ролей.

Сейчас роли работают как точное совпадение:

role:editor

означает:

У пользователя должна быть именно роль editor.

Но в реальной жизни должно быть умнее:

owner  — самый главный
admin  — ниже owner
editor — ниже admin
viewer — только просмотр

То есть если у пользователя роль owner, он автоматически должен проходить проверки:

role:admin
role:editor
role:viewer

Потому что owner выше всех.


---

1. Замени /local/mvc/Core/Role.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * Role
 *
 * Проверка проектных ролей.
 *
 * Поддерживает иерархию:
 * owner > admin > editor > viewer
 */
class Role
{
    private static ?RoleResolverInterface $resolver = null;

    /**
     * Проверить роль с учётом иерархии.
     *
     * Например:
     * если у пользователя owner,
     * то Role::has('editor') вернёт true.
     */
    public static function has(string $role, ?int $userId = null): bool
    {
        $role = trim($role);

        if ($role === '') {
            return false;
        }

        $userRoles = self::all($userId);

        if (empty($userRoles)) {
            return false;
        }

        /**
         * Сначала проверяем точное совпадение.
         */
        if (in_array($role, $userRoles, true)) {
            return true;
        }

        /**
         * Потом проверяем по иерархии.
         */
        $requiredRank = self::rank($role);

        if ($requiredRank <= 0) {
            return false;
        }

        foreach ($userRoles as $userRole) {
            if (self::rank((string)$userRole) >= $requiredRank) {
                return true;
            }
        }

        return false;
    }

    /**
     * Проверить, есть ли хотя бы одна роль из списка.
     */
    public static function hasAny(array $roles, ?int $userId = null): bool
    {
        foreach ($roles as $role) {
            if (self::has((string)$role, $userId)) {
                return true;
            }
        }

        return false;
    }

    /**
     * Получить роли пользователя.
     */
    public static function all(?int $userId = null): array
    {
        return self::resolver()->roles($userId);
    }

    /**
     * Получить числовой уровень роли.
     *
     * Чем больше число — тем выше роль.
     */
    public static function rank(string $role): int
    {
        $role = trim($role);

        $hierarchy = self::hierarchy();

        return (int)($hierarchy[$role] ?? 0);
    }

    /**
     * Роли из config.php.
     */
    public static function hierarchy(): array
    {
        $hierarchy = Config::get('roles.hierarchy', [
            'viewer' => 10,
            'editor' => 20,
            'admin' => 30,
            'owner' => 40,
        ]);

        return is_array($hierarchy) ? $hierarchy : [];
    }

    /**
     * Самая высокая роль пользователя.
     */
    public static function highest(?int $userId = null): string
    {
        $roles = self::all($userId);

        $highestRole = '';
        $highestRank = 0;

        foreach ($roles as $role) {
            $rank = self::rank((string)$role);

            if ($rank > $highestRank) {
                $highestRank = $rank;
                $highestRole = (string)$role;
            }
        }

        return $highestRole;
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

2. Обнови /local/mvc_demo/config.php

Добавь иерархию ролей в блок roles.

Должно быть так:

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

        /**
         * Чем больше число — тем выше роль.
         */
        'hierarchy' => [
            'viewer' => 10,
            'editor' => 20,
            'admin' => 30,
            'owner' => 40,
        ],
    ],
];


---

3. Обнови /local/mvc_demo/Services/DemoRoleResolver.php

Сейчас лучше сделать так, чтобы админ получал только owner, а не все роли сразу. Тогда мы точно проверим, что иерархия работает.

<?php

namespace Local\MvcDemo\Services;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\RoleResolverInterface;

/**
 * DemoRoleResolver
 *
 * Тестовая проверка ролей для mvc_demo.
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
         * админ Битрикса получает только owner.
         *
         * Благодаря иерархии owner проходит проверки:
         * role:admin
         * role:editor
         * role:viewer
         */
        if (Auth::isAdmin()) {
            return [
                'owner',
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

4. Обнови тестовые методы в HomeController

Файл:

/local/mvc_demo/Controllers/HomeController.php

Добавь метод:

public function roleInfo(): Response
{
    return $this->success([
        'roles' => \Local\Mvc\Core\Role::all(),
        'highest' => \Local\Mvc\Core\Role::highest(),
        'checks' => [
            'viewer' => \Local\Mvc\Core\Role::has('viewer'),
            'editor' => \Local\Mvc\Core\Role::has('editor'),
            'admin' => \Local\Mvc\Core\Role::has('admin'),
            'owner' => \Local\Mvc\Core\Role::has('owner'),
        ],
        'hierarchy' => \Local\Mvc\Core\Role::hierarchy(),
    ]);
}


---

5. Добавь маршрут в /local/mvc_demo/routes.php

$router->get('/role-info', [HomeController::class, 'roleInfo'], ['auth']);

Рядом с тестами ролей можно сделать так:

$router->get('/role-info', [HomeController::class, 'roleInfo'], ['auth']);

$router->get('/role-viewer-test', [HomeController::class, 'roleViewerTest'], ['auth', 'role:viewer']);

$router->get('/role-editor-test', [HomeController::class, 'roleEditorTest'], ['auth', 'role:editor']);


---

6. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/role-info

Если ты админ Битрикса, должно быть примерно так:

{
  "ok": true,
  "data": {
    "roles": ["owner"],
    "highest": "owner",
    "checks": {
      "viewer": true,
      "editor": true,
      "admin": true,
      "owner": true
    }
  }
}

Вот это главное:

roles: ["owner"]
но editor/admin/viewer тоже true

Значит иерархия работает.

Потом проверь:

https://bitrix24-stage.gaz.ru/local/mvc_demo/role-editor-test

Если у тебя роль owner, маршрут role:editor должен пустить.


---

Что мы сделали

Раньше:

role:editor

означало:

нужна именно editor

Теперь:

role:editor

означает:

нужна editor или выше

Иерархия:

owner  = 40
admin  = 30
editor = 20
viewer = 10

То есть:

owner проходит всё
admin проходит admin/editor/viewer
editor проходит editor/viewer
viewer проходит только viewer

Это очень важно для будущего sitebuilder, потому что там будет удобно писать:

$router->get('/sites/{siteId}/edit', [SiteController::class, 'edit'], ['auth', 'role:editor']);

$router->post('/sites/{siteId}/delete', [SiteController::class, 'delete'], ['auth', 'role:owner', 'csrf']);

Контроллер не будет сам разбираться, кто выше. Это делает фреймворк.