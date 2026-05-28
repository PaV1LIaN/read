Идём дальше. Сейчас сделаем важную вещь для реальных проектов: роли с контекстом.

Сейчас у нас роль проверяется просто так:

role:editor

Но для sitebuilder этого мало.

Почему?

Потому что пользователь может быть:

editor на сайте 5
viewer на сайте 7
owner на сайте 10

То есть роль зависит не просто от пользователя, а от конкретного объекта.

Например:

/local/sitebuilder/sites/5/edit

Здесь надо проверить:

А у пользователя есть роль editor именно на site_id = 5?


---

Что сделаем

Мы научим роли получать контекст маршрута.

Например маршрут:

$router->get('/site-role-test/{siteId:\d+}', [HomeController::class, 'siteRoleTest'], ['auth', 'role:editor']);

Если открыть:

/local/mvc_demo/site-role-test/5

то middleware сможет передать в Role:

[
    'route' => [
        'siteId' => 5
    ]
]


---

1. Замени /local/mvc/Core/RoleResolverInterface.php

<?php

namespace Local\Mvc\Core;

/**
 * RoleResolverInterface
 *
 * Интерфейс проверки проектных ролей.
 *
 * Важно:
 * $context нужен, чтобы проверять роль не просто глобально,
 * а относительно конкретного объекта.
 *
 * Например:
 * siteId = 5
 * pageId = 10
 */
interface RoleResolverInterface
{
    public function hasRole(string $role, ?int $userId = null, array $context = []): bool;

    public function roles(?int $userId = null, array $context = []): array;
}


---

2. Замени /local/mvc/Core/Role.php

<?php

namespace Local\Mvc\Core;

use RuntimeException;

/**
 * Role
 *
 * Проверка проектных ролей.
 *
 * Поддерживает:
 * - роли проекта
 * - иерархию ролей
 * - контекст маршрута
 */
class Role
{
    private static ?RoleResolverInterface $resolver = null;

    /**
     * Проверить роль с учётом иерархии.
     */
    public static function has(string $role, ?int $userId = null, array $context = []): bool
    {
        $role = trim($role);

        if ($role === '') {
            return false;
        }

        $userRoles = self::all($userId, $context);

        if (empty($userRoles)) {
            return false;
        }

        if (in_array($role, $userRoles, true)) {
            return true;
        }

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

    public static function hasAny(array $roles, ?int $userId = null, array $context = []): bool
    {
        foreach ($roles as $role) {
            if (self::has((string)$role, $userId, $context)) {
                return true;
            }
        }

        return false;
    }

    public static function all(?int $userId = null, array $context = []): array
    {
        return self::resolver()->roles($userId, $context);
    }

    public static function highest(?int $userId = null, array $context = []): string
    {
        $roles = self::all($userId, $context);

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

    public static function rank(string $role): int
    {
        $role = trim($role);

        $hierarchy = self::hierarchy();

        return (int)($hierarchy[$role] ?? 0);
    }

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

Найди блок:

if ($name === 'role') {

И замени его на:

if ($name === 'role') {
    $context = self::contextFromRequest($request);

    if ($argument !== '' && Role::has($argument, null, $context)) {
        return null;
    }

    return Response::json([
        'ok' => false,
        'error' => 'ROLE_REQUIRED',
        'details' => [
            'message' => 'Недостаточно прав. Требуется роль: ' . $argument,
            'required_role' => $argument,
            'user_roles' => Role::all(null, $context),
            'context' => $context,
        ],
    ], 403);
}

Найди блок:

if ($name === 'roles') {

И замени его на:

if ($name === 'roles') {
    $context = self::contextFromRequest($request);
    $requiredRoles = self::parseRoleList($argument);

    if (Role::hasAny($requiredRoles, null, $context)) {
        return null;
    }

    return Response::json([
        'ok' => false,
        'error' => 'ROLES_REQUIRED',
        'details' => [
            'message' => 'Недостаточно прав. Требуется одна из ролей.',
            'required_roles' => $requiredRoles,
            'user_roles' => Role::all(null, $context),
            'context' => $context,
        ],
    ], 403);
}

В конец класса Middleware, рядом с parseRoleList(), добавь метод:

/**
 * Собрать контекст для проверки ролей.
 *
 * Сюда кладём параметры маршрута.
 *
 * Например:
 * /site-role-test/{siteId}
 *
 * даст:
 * [
 *   'route' => [
 *     'siteId' => 5
 *   ]
 * ]
 */
private static function contextFromRequest(Request $request): array
{
    return [
        'route' => $request->routeParams(),
        'method' => $request->method(),
        'path' => $request->path(),
    ];
}


---

4. Замени /local/mvc_demo/Services/DemoRoleResolver.php

<?php

namespace Local\MvcDemo\Services;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\RoleResolverInterface;

/**
 * DemoRoleResolver
 *
 * Тестовая проверка ролей.
 *
 * В реальном проекте здесь можно смотреть:
 * - siteId
 * - pageId
 * - applicationId
 * - права из таблицы
 */
class DemoRoleResolver implements RoleResolverInterface
{
    public function hasRole(string $role, ?int $userId = null, array $context = []): bool
    {
        return in_array($role, $this->roles($userId, $context), true);
    }

    public function roles(?int $userId = null, array $context = []): array
    {
        if (!Auth::check()) {
            return [];
        }

        /**
         * Для demo смотрим siteId из маршрута.
         *
         * /site-role-test/1
         * /site-role-test/2
         * /site-role-test/3
         */
        $siteId = (int)($context['route']['siteId'] ?? 0);

        /**
         * Админ Битрикса всегда owner.
         */
        if (Auth::isAdmin()) {
            return ['owner'];
        }

        /**
         * Тестовая логика для обычного пользователя:
         *
         * siteId = 1 => viewer
         * siteId = 2 => editor
         * siteId = 3 => admin
         * другое     => viewer
         */
        if ($siteId === 3) {
            return ['admin'];
        }

        if ($siteId === 2) {
            return ['editor'];
        }

        return ['viewer'];
    }
}


---

5. Добавь метод в /local/mvc_demo/Controllers/HomeController.php

Внутрь класса добавь:

public function siteRoleTest(string $siteId): Response
{
    return $this->success([
        'message' => 'Доступ к site-role-test разрешён.',
        'site_id' => (int)$siteId,
        'roles' => \Local\Mvc\Core\Role::all(null, [
            'route' => [
                'siteId' => (int)$siteId,
            ],
        ]),
        'highest' => \Local\Mvc\Core\Role::highest(null, [
            'route' => [
                'siteId' => (int)$siteId,
            ],
        ]),
    ]);
}


---

6. Добавь маршрут в /local/mvc_demo/routes.php

Добавь рядом с тестами ролей:

$router->get('/site-role-test/{siteId:\d+}', [HomeController::class, 'siteRoleTest'], ['auth', 'role:editor']);


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/site-role-test/1

Для обычного пользователя там будет viewer, а нужен editor, значит должно быть 403.

Открой:

/local/mvc_demo/site-role-test/2

Там обычный пользователь получает editor, значит доступ разрешён.

Открой:

/local/mvc_demo/site-role-test/3

Там обычный пользователь получает admin, а admin выше editor, значит доступ тоже разрешён.

Если ты админ Битрикса, везде будет доступ, потому что админ получает owner.


---

Что мы сделали

Раньше роль была просто ролью пользователя:

user → role

Теперь роль может зависеть от объекта:

user + siteId → role

Это очень важно.

Для sitebuilder потом будет так:

$router->get('/sites/{siteId:\d+}/edit', [SiteController::class, 'edit'], ['auth', 'role:editor']);

И внутри resolver можно будет проверить:

Какую роль пользователь имеет именно на этом сайте?

Например:

site_id = 5
user_id = 101
role = editor

Главная мысль:

Middleware достаёт siteId из маршрута.
Role передаёт siteId в resolver.
Resolver решает, есть ли доступ.

Вот это уже похоже на нормальный фундамент для sitebuilder, glab и других проектов.