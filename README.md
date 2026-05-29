Идём дальше в Laravel-стиле. Следующий важный шаг — middleware как отдельные классы, а не один большой файл.

Сейчас у нас вся логика лежит в одном файле:

/local/mvc/Core/Middleware.php

Там сразу всё:

auth
admin
csrf
group
role

А в Laravel middleware — это отдельные классы:

AuthMiddleware
AdminMiddleware
CsrfMiddleware
RoleMiddleware

Так проще расширять фреймворк.


---

Что хотим получить

В маршрутах оставляем красиво:

Route::get('/admin/users', [AdminController::class, 'users'])
    ->middleware(['auth', 'admin']);

Но внутри фреймворка auth будет ссылаться на класс:

Local\Mvc\Core\Middlewares\AuthMiddleware::class

То есть:

auth  → AuthMiddleware
admin → AdminMiddleware
csrf  → CsrfMiddleware
role  → RoleMiddleware


---

1. Создай /local/mvc/Core/MiddlewareInterface.php

<?php

namespace Local\Mvc\Core;

/**
 * MiddlewareInterface
 *
 * Интерфейс для middleware-классов.
 *
 * Каждый middleware получает Request
 * и может вернуть Response, если надо остановить запрос.
 *
 * Если вернул null — запрос идёт дальше.
 */
interface MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response;
}


---

2. Создай папку /local/mvc/Core/Middlewares/

/local/mvc/Core/Middlewares/


---

3. Создай /local/mvc/Core/Middlewares/AuthMiddleware.php

<?php

namespace Local\Mvc\Core\Middlewares;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\MiddlewareInterface;
use Local\Mvc\Core\Request;
use Local\Mvc\Core\Response;

class AuthMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response
    {
        if (Auth::check()) {
            return null;
        }

        return Response::json([
            'ok' => false,
            'error' => 'AUTH_REQUIRED',
            'details' => [
                'message' => 'Нужно авторизоваться',
            ],
        ], 401);
    }
}


---

4. Создай /local/mvc/Core/Middlewares/AdminMiddleware.php

<?php

namespace Local\Mvc\Core\Middlewares;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\MiddlewareInterface;
use Local\Mvc\Core\Request;
use Local\Mvc\Core\Response;

class AdminMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response
    {
        if (Auth::isAdmin()) {
            return null;
        }

        return Response::json([
            'ok' => false,
            'error' => 'ADMIN_REQUIRED',
            'details' => [
                'message' => 'Нужны права администратора',
            ],
        ], 403);
    }
}


---

5. Создай /local/mvc/Core/Middlewares/CsrfMiddleware.php

<?php

namespace Local\Mvc\Core\Middlewares;

use Local\Mvc\Core\MiddlewareInterface;
use Local\Mvc\Core\Request;
use Local\Mvc\Core\Response;

class CsrfMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response
    {
        /**
         * Безопасные методы не проверяем.
         */
        if (in_array($request->method(), ['GET', 'HEAD', 'OPTIONS'], true)) {
            return null;
        }

        /**
         * Обычная форма Битрикса через bitrix_sessid_post().
         */
        if (function_exists('check_bitrix_sessid') && check_bitrix_sessid()) {
            return null;
        }

        /**
         * AJAX/JSON-запрос через заголовок.
         */
        $headerSessid = (string)$request->header('X-Bitrix-Sessid', '');

        if (
            $headerSessid !== ''
            && function_exists('bitrix_sessid')
            && hash_equals((string)bitrix_sessid(), $headerSessid)
        ) {
            return null;
        }

        return Response::json([
            'ok' => false,
            'error' => 'BAD_SESSID',
            'details' => [
                'message' => 'Неверный sessid. Обновите страницу и попробуйте снова.',
            ],
        ], 403);
    }
}


---

6. Создай /local/mvc/Core/Middlewares/GroupMiddleware.php

<?php

namespace Local\Mvc\Core\Middlewares;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\MiddlewareInterface;
use Local\Mvc\Core\Request;
use Local\Mvc\Core\Response;

class GroupMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response
    {
        $groupId = (int)$argument;

        if ($groupId > 0 && Auth::inGroup($groupId)) {
            return null;
        }

        return Response::json([
            'ok' => false,
            'error' => 'GROUP_REQUIRED',
            'details' => [
                'message' => 'Недостаточно прав. Требуется группа: ' . $groupId,
                'required_group' => $groupId,
                'user_groups' => Auth::groups(),
            ],
        ], 403);
    }
}


---

7. Создай /local/mvc/Core/Middlewares/GroupsMiddleware.php

<?php

namespace Local\Mvc\Core\Middlewares;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\MiddlewareInterface;
use Local\Mvc\Core\Request;
use Local\Mvc\Core\Response;

class GroupsMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response
    {
        $requiredGroups = $this->parseGroupList($argument);
        $userGroups = Auth::groups();

        foreach ($requiredGroups as $groupId) {
            if (in_array($groupId, $userGroups, true)) {
                return null;
            }
        }

        return Response::json([
            'ok' => false,
            'error' => 'GROUPS_REQUIRED',
            'details' => [
                'message' => 'Недостаточно прав. Требуется одна из групп.',
                'required_groups' => $requiredGroups,
                'user_groups' => $userGroups,
            ],
        ], 403);
    }

    private function parseGroupList(string $argument): array
    {
        $items = explode(',', $argument);
        $groups = [];

        foreach ($items as $item) {
            $groupId = (int)trim($item);

            if ($groupId > 0) {
                $groups[] = $groupId;
            }
        }

        return array_values(array_unique($groups));
    }
}


---

8. Создай /local/mvc/Core/Middlewares/RoleMiddleware.php

<?php

namespace Local\Mvc\Core\Middlewares;

use Local\Mvc\Core\MiddlewareInterface;
use Local\Mvc\Core\Request;
use Local\Mvc\Core\Response;
use Local\Mvc\Core\Role;

class RoleMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response
    {
        $context = $this->contextFromRequest($request);

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

    private function contextFromRequest(Request $request): array
    {
        return [
            'route' => $request->routeParams(),
            'method' => $request->method(),
            'path' => $request->path(),
        ];
    }
}


---

9. Создай /local/mvc/Core/Middlewares/RolesMiddleware.php

<?php

namespace Local\Mvc\Core\Middlewares;

use Local\Mvc\Core\MiddlewareInterface;
use Local\Mvc\Core\Request;
use Local\Mvc\Core\Response;
use Local\Mvc\Core\Role;

class RolesMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, string $argument = ''): ?Response
    {
        $context = $this->contextFromRequest($request);
        $requiredRoles = $this->parseRoleList($argument);

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

    private function parseRoleList(string $argument): array
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

    private function contextFromRequest(Request $request): array
    {
        return [
            'route' => $request->routeParams(),
            'method' => $request->method(),
            'path' => $request->path(),
        ];
    }
}


---

10. Обнови /local/mvc/Core/App.php

В методе loadConfig() найди блок:

Config::load([
    'app' => [
        'name' => 'Local MVC App',
        'description' => '',
    ],
    'debug' => defined('LOCAL_MVC_DEBUG') && LOCAL_MVC_DEBUG === true,
    'log' => [
        'file' => rtrim($projectRoot, '/') . '/logs/app.log',
    ],
]);

Замени на:

Config::load([
    'app' => [
        'name' => 'Local MVC App',
        'description' => '',
    ],

    'debug' => defined('LOCAL_MVC_DEBUG') && LOCAL_MVC_DEBUG === true,

    'log' => [
        'file' => rtrim($projectRoot, '/') . '/logs/app.log',
    ],

    /**
     * Laravel-like aliases middleware.
     */
    'middleware' => [
        'auth' => \Local\Mvc\Core\Middlewares\AuthMiddleware::class,
        'admin' => \Local\Mvc\Core\Middlewares\AdminMiddleware::class,
        'csrf' => \Local\Mvc\Core\Middlewares\CsrfMiddleware::class,

        'group' => \Local\Mvc\Core\Middlewares\GroupMiddleware::class,
        'groups' => \Local\Mvc\Core\Middlewares\GroupsMiddleware::class,

        'role' => \Local\Mvc\Core\Middlewares\RoleMiddleware::class,
        'roles' => \Local\Mvc\Core\Middlewares\RolesMiddleware::class,
    ],
]);


---

11. Замени /local/mvc/Core/Middleware.php

Теперь этот файл будет не хранить всю логику, а только запускать нужные middleware-классы.

<?php

namespace Local\Mvc\Core;

/**
 * Middleware
 *
 * Dispatcher middleware.
 *
 * Простыми словами:
 * принимает строки:
 *
 * auth
 * admin
 * role:editor
 *
 * находит нужный класс
 * и запускает его.
 */
class Middleware
{
    public static function handle(array $middlewares, Request $request): ?Response
    {
        foreach ($middlewares as $middleware) {
            $middleware = trim((string)$middleware);

            if ($middleware === '') {
                continue;
            }

            $response = self::handleOne($middleware, $request);

            if ($response instanceof Response) {
                return $response;
            }
        }

        return null;
    }

    private static function handleOne(string $middleware, Request $request): ?Response
    {
        [$name, $argument] = self::parse($middleware);

        $class = self::resolveMiddlewareClass($name);

        if ($class === '') {
            return Response::json([
                'ok' => false,
                'error' => 'UNKNOWN_MIDDLEWARE',
                'details' => [
                    'middleware' => $middleware,
                    'name' => $name,
                ],
            ], 500);
        }

        $instance = App::container()->make($class);

        if (!$instance instanceof MiddlewareInterface) {
            return Response::json([
                'ok' => false,
                'error' => 'INVALID_MIDDLEWARE',
                'details' => [
                    'middleware' => $middleware,
                    'class' => $class,
                    'message' => 'Middleware должен реализовывать MiddlewareInterface.',
                ],
            ], 500);
        }

        return $instance->handle($request, $argument);
    }

    private static function resolveMiddlewareClass(string $name): string
    {
        $aliases = Config::get('middleware', []);

        if (!is_array($aliases)) {
            return '';
        }

        return (string)($aliases[$name] ?? '');
    }

    private static function parse(string $middleware): array
    {
        $parts = explode(':', $middleware, 2);

        $name = trim((string)($parts[0] ?? ''));
        $argument = trim((string)($parts[1] ?? ''));

        return [$name, $argument];
    }
}


---

12. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь всё, где используются middleware:

/local/mvc_demo/admin/dashboard
/local/mvc_demo/admin/users
/local/mvc_demo/form
/local/mvc_demo/method-demo
/local/mvc_demo/role-info
/local/mvc_demo/role-editor-test
/local/mvc_demo/site-role-test/2
/local/mvc_demo/api/users
/local/mvc_demo/debug/routes


---

Что мы сделали

Раньше Middleware.php был большим файлом со всей логикой.

Теперь стало Laravel-like:

auth   → AuthMiddleware
admin  → AdminMiddleware
csrf   → CsrfMiddleware
role   → RoleMiddleware
group  → GroupMiddleware

В маршрутах ничего не изменилось:

Route::get('/users', [AdminController::class, 'users'])
    ->middleware(['auth', 'admin']);

Но внутри теперь архитектура ближе к Laravel.


---

Почему это важно

Теперь в любом проекте можно добавить свой middleware.

Например для sitebuilder:

'can.edit.site' => \Local\Sitebuilder\Middlewares\CanEditSiteMiddleware::class,

И потом в маршруте:

Route::get('/sites/{siteId:\d+}/edit', [SiteController::class, 'edit'])
    ->middleware(['auth', 'can.edit.site'])
    ->name('sites.edit');

То есть фреймворк становится расширяемым, как Laravel.