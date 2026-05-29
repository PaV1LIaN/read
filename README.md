Отлично, идём дальше. Сейчас сделаем автоматические префиксы имён маршрутов.

Сейчас мы пишем так:

$router->get('/dashboard', [AdminController::class, 'dashboard'], [], 'admin.dashboard');

$router->get('/users', [AdminController::class, 'users'], [], 'admin.users.index');

$router->get('/users/{id:\d+}', [AdminController::class, 'userDetail'], [], 'admin.users.show');

Работает, но длинно.

Хотим так:

$router->group([
    'prefix' => '/admin',
    'middleware' => ['admin'],
    'as' => 'admin.',
], function (Router $router) {
    $router->get('/dashboard', [AdminController::class, 'dashboard'], [], 'dashboard');
    $router->get('/users', [AdminController::class, 'users'], [], 'users.index');
    $router->get('/users/{id:\d+}', [AdminController::class, 'userDetail'], [], 'users.show');
});

А итоговые имена будут:

admin.dashboard
admin.users.index
admin.users.show

То есть группа сама добавляет начало имени.


---

1. Замени /local/mvc/Core/Router.php

<?php

namespace Local\Mvc\Core;

/**
 * Router
 *
 * Диспетчер маршрутов.
 */
class Router
{
    private array $routes = [];

    /**
     * Маршруты по имени.
     */
    private array $namedRoutes = [];

    private string $groupPrefix = '';

    private array $groupMiddleware = [];

    /**
     * Префикс имён маршрутов внутри группы.
     *
     * Например:
     * admin.
     * api.
     */
    private string $groupNamePrefix = '';

    public function get(string $path, array $handler, array $middleware = [], ?string $name = null): void
    {
        $this->add('GET', $path, $handler, $middleware, $name);
    }

    public function post(string $path, array $handler, array $middleware = [], ?string $name = null): void
    {
        $this->add('POST', $path, $handler, $middleware, $name);
    }

    /**
     * Получить список всех маршрутов.
     */
    public function routes(): array
    {
        $list = [];

        foreach ($this->routes as $method => $routes) {
            foreach ($routes as $route) {
                $handler = $route['handler'] ?? [];

                $controller = $handler[0] ?? '';
                $action = $handler[1] ?? '';

                $list[] = [
                    'method' => $method,
                    'path' => $route['path'] ?? '',
                    'name' => $route['name'] ?? '',
                    'controller' => (string)$controller,
                    'action' => (string)$action,
                    'middleware' => $route['middleware'] ?? [],
                ];
            }
        }

        return $list;
    }

    /**
     * Группа маршрутов.
     *
     * Поддерживает:
     * prefix     => /admin
     * middleware => ['auth', 'admin']
     * as         => admin.
     */
    public function group(array $options, callable $callback): void
    {
        $oldPrefix = $this->groupPrefix;
        $oldMiddleware = $this->groupMiddleware;
        $oldNamePrefix = $this->groupNamePrefix;

        $prefix = (string)($options['prefix'] ?? '');
        $namePrefix = (string)($options['as'] ?? '');
        $middleware = $options['middleware'] ?? [];

        if (!is_array($middleware)) {
            $middleware = [$middleware];
        }

        if ($prefix !== '') {
            $this->groupPrefix = $this->joinPaths($this->groupPrefix, $prefix);
        }

        if ($namePrefix !== '') {
            $this->groupNamePrefix .= $namePrefix;
        }

        $this->groupMiddleware = array_values(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        )));

        $callback($this);

        $this->groupPrefix = $oldPrefix;
        $this->groupMiddleware = $oldMiddleware;
        $this->groupNamePrefix = $oldNamePrefix;
    }

    private function add(string $method, string $path, array $handler, array $middleware = [], ?string $name = null): void
    {
        $method = strtoupper($method);

        $path = $this->joinPaths($this->groupPrefix, $path);
        $path = $this->normalizePath($path);

        $middleware = array_values(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        )));

        /**
         * Если маршрут внутри группы с "as",
         * добавляем префикс имени.
         *
         * Было:
         * users.show
         *
         * В группе:
         * as => admin.
         *
         * Стало:
         * admin.users.show
         */
        $finalName = null;

        if ($name !== null && $name !== '') {
            $finalName = $this->groupNamePrefix . $name;
        }

        $compiled = $this->compilePath($path);

        $route = [
            'method' => $method,
            'path' => $path,
            'pattern' => $compiled['pattern'],
            'params' => $compiled['params'],
            'handler' => $handler,
            'middleware' => $middleware,
            'name' => $finalName,
        ];

        $this->routes[$method][] = $route;

        if ($finalName !== null && $finalName !== '') {
            $this->namedRoutes[$finalName] = $route;
        }
    }

    /**
     * Собрать URL по имени маршрута.
     */
    public function url(string $name, array $params = [], array $query = []): string
    {
        if (!isset($this->namedRoutes[$name])) {
            return '#route-not-found-' . rawurlencode($name);
        }

        $route = $this->namedRoutes[$name];

        $path = (string)($route['path'] ?? '/');

        $path = preg_replace_callback(
            '#\{([a-zA-Z_][a-zA-Z0-9_]*)(?::[^}]+)?\}#',
            static function ($matches) use ($params) {
                $key = $matches[1];

                return rawurlencode((string)($params[$key] ?? ''));
            },
            $path
        );

        $url = rtrim(App::projectUrl(), '/') . $path;

        if (!empty($query)) {
            $url .= '?' . http_build_query($query);
        }

        return $url;
    }

    public function dispatch(Request $request): void
    {
        $method = $request->method();
        $path = $this->normalizePath($request->path());

        $matched = $this->match($method, $path);

        if ($matched === null) {
            $this->notFound($method, $path);
            return;
        }

        $route = $matched['route'];
        $routeParams = $matched['params'];

        $request->setRouteParams($routeParams);

        $handler = $route['handler'] ?? [];
        $middlewares = $route['middleware'] ?? [];

        $middlewareResponse = Middleware::handle($middlewares, $request);

        if ($middlewareResponse instanceof Response) {
            $middlewareResponse->send();
            return;
        }

        $controllerClass = $handler[0] ?? null;
        $controllerMethod = $handler[1] ?? null;

        if (!$controllerClass || !class_exists($controllerClass)) {
            $this->serverError('Контроллер не найден: ' . (string)$controllerClass);
            return;
        }

        $controller = new $controllerClass($request);

        if (!$controllerMethod || !method_exists($controller, $controllerMethod)) {
            $methods = get_class_methods($controller);

            $this->serverError(
                'Метод контроллера не найден: ' . $controllerClass . '::' . (string)$controllerMethod
                . "\n\nPHP видит такие методы:\n"
                . implode("\n", $methods)
            );

            return;
        }

        $result = $controller->{$controllerMethod}(...array_values($routeParams));

        if ($result instanceof Response) {
            $result->send();
            return;
        }
    }

    private function match(string $method, string $path): ?array
    {
        $routes = $this->routes[$method] ?? [];

        foreach ($routes as $route) {
            $matches = [];

            if (!preg_match($route['pattern'], $path, $matches)) {
                continue;
            }

            $params = [];

            foreach (($route['params'] ?? []) as $paramName) {
                $params[$paramName] = $matches[$paramName] ?? null;
            }

            return [
                'route' => $route,
                'params' => $params,
            ];
        }

        return null;
    }

    private function compilePath(string $path): array
    {
        $params = [];
        $pattern = '';
        $offset = 0;

        preg_match_all(
            '#\{([a-zA-Z_][a-zA-Z0-9_]*)(?::([^}]+))?\}#',
            $path,
            $matches,
            PREG_OFFSET_CAPTURE
        );

        foreach ($matches[0] as $index => $match) {
            $full = $match[0];
            $position = $match[1];

            $staticPart = substr($path, $offset, $position - $offset);
            $pattern .= preg_quote($staticPart, '#');

            $name = $matches[1][$index][0];
            $rule = $matches[2][$index][0] ?? '[^/]+';
            $rule = str_replace('#', '\#', $rule);

            $params[] = $name;

            $pattern .= '(?P<' . $name . '>' . $rule . ')';

            $offset = $position + strlen($full);
        }

        $pattern .= preg_quote(substr($path, $offset), '#');

        return [
            'pattern' => '#^' . $pattern . '$#u',
            'params' => $params,
        ];
    }

    private function joinPaths(string $left, string $right): string
    {
        $left = trim($left);
        $right = trim($right);

        if ($left === '' && $right === '') {
            return '/';
        }

        if ($left === '') {
            return $this->normalizePath($right);
        }

        if ($right === '') {
            return $this->normalizePath($left);
        }

        return $this->normalizePath(trim($left, '/') . '/' . trim($right, '/'));
    }

    private function normalizePath(string $path): string
    {
        $path = trim($path);

        if ($path === '') {
            return '/';
        }

        $path = '/' . trim($path, '/');

        if ($path !== '/') {
            $path = rtrim($path, '/');
        }

        return $path;
    }

    private function notFound(string $method, string $path): void
    {
        Response::html(
            '<h1>404</h1>'
            . '<p>Маршрут не найден.</p>'
            . '<pre>'
            . 'Method: ' . htmlspecialchars($method) . "\n"
            . 'Path: ' . htmlspecialchars($path) . "\n"
            . '</pre>',
            404
        )->send();

        exit;
    }

    private function serverError(string $message): void
    {
        Response::html(
            '<h1>500</h1>'
            . '<p>Ошибка MVC.</p>'
            . '<pre>' . htmlspecialchars($message) . '</pre>',
            500
        )->send();

        exit;
    }
}


---

2. Обнови /local/mvc_demo/routes.php

Теперь можно упростить имена в группах.

Админская группа

Было:

$router->group([
    'prefix' => '/admin',
    'middleware' => ['admin'],
], function (Router $router) {
    $router->get('/dashboard', [AdminController::class, 'dashboard'], [], 'admin.dashboard');

    $router->get('/users', [AdminController::class, 'users'], [], 'admin.users.index');

    $router->get('/users/{id:\d+}', [AdminController::class, 'userDetail'], [], 'admin.users.show');
});

Сделай так:

$router->group([
    'prefix' => '/admin',
    'middleware' => ['admin'],
    'as' => 'admin.',
], function (Router $router) {
    $router->get('/dashboard', [AdminController::class, 'dashboard'], [], 'dashboard');

    $router->get('/users', [AdminController::class, 'users'], [], 'users.index');

    $router->get('/users/{id:\d+}', [AdminController::class, 'userDetail'], [], 'users.show');
});

Имена в итоге останутся те же:

admin.dashboard
admin.users.index
admin.users.show

Поэтому ссылки mvc_route('admin.users.show') продолжат работать.


---

API-группа

Было можно без имён. Но теперь добавим красиво:

$router->group([
    'prefix' => '/api',
    'middleware' => ['auth', 'admin'],
    'as' => 'api.',
], function (Router $router) {
    $router->get('/users', [UserApiController::class, 'index'], [], 'users.index');

    $router->get('/users/{id:\d+}', [UserApiController::class, 'show'], [], 'users.show');

    $router->post('/ajax-demo/echo', [AjaxDemoController::class, 'echoText'], ['csrf'], 'ajax.echo');

    $router->get('/error-test', [AjaxDemoController::class, 'errorTest'], [], 'error.test');
});

Получатся имена:

api.users.index
api.users.show
api.ajax.echo
api.error.test


---

3. Проверь debug/routes

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/debug/routes

В колонке Имя у маршрутов должны появиться:

admin.dashboard
admin.users.index
admin.users.show
api.users.index
api.users.show
api.ajax.echo
api.error.test


---

Что мы сделали

Раньше каждый маршрут внутри группы должен был писать полное имя:

'admin.users.show'

Теперь группа добавляет начало сама:

'as' => 'admin.'

А маршрут пишет только свою часть:

'users.show'

Итог:

admin. + users.show = admin.users.show

Главная мысль:

prefix — добавляет начало URL
as     — добавляет начало имени маршрута

Пример:

$router->group([
    'prefix' => '/sites',
    'as' => 'sites.',
], function (Router $router) {
    $router->get('/', [SiteController::class, 'index'], [], 'index');
    $router->get('/{siteId:\d+}/edit', [SiteController::class, 'edit'], [], 'edit');
});

Получится:

URL:  /sites
Name: sites.index

URL:  /sites/{siteId}/edit
Name: sites.edit