Отлично, тогда с этого момента держим курс: наш MVC должен быть похож на Laravel по подходу и синтаксису, насколько это нормально для Битрикс24.

То есть будем постепенно двигаться к таким вещам:

Route::get('/users', [UserController::class, 'index'])
    ->middleware(['auth'])
    ->name('users.index');

Route::prefix('/admin')
    ->middleware(['auth', 'admin'])
    ->name('admin.')
    ->group(function () {
        Route::get('/users', [AdminController::class, 'users'])
            ->name('users.index');
    });

Это как в Laravel: маршруты читаются легко, контроллеры тонкие, логика в сервисах, модели отдельно.


---

Что делаем сейчас

Сделаем Laravel-like Route facade.

Чтобы вместо:

$router->get('/form', [FormController::class, 'index'], [], 'form.index');

можно было писать:

Route::get('/form', [FormController::class, 'index'])
    ->name('form.index');


---

1. Создай /local/mvc/Core/RouteDefinition.php

<?php

namespace Local\Mvc\Core;

/**
 * RouteDefinition
 *
 * Объект одного маршрута.
 *
 * Нужен, чтобы писать почти как в Laravel:
 *
 * Route::get('/users', [UserController::class, 'index'])
 *     ->middleware(['auth'])
 *     ->name('users.index');
 */
class RouteDefinition
{
    public function __construct(
        private Router $router,
        private string $method,
        private int $index
    ) {}

    public function name(string $name): self
    {
        $this->router->setRouteName($this->method, $this->index, $name);

        return $this;
    }

    public function middleware(array|string $middleware): self
    {
        $this->router->addRouteMiddleware($this->method, $this->index, $middleware);

        return $this;
    }
}


---

2. Замени /local/mvc/Core/Router.php

<?php

namespace Local\Mvc\Core;

/**
 * Router
 *
 * Диспетчер маршрутов.
 *
 * Теперь поддерживает Laravel-like синтаксис:
 *
 * Route::get('/path', [Controller::class, 'method'])
 *     ->middleware(['auth'])
 *     ->name('route.name');
 */
class Router
{
    private array $routes = [];

    private array $namedRoutes = [];

    private string $groupPrefix = '';

    private array $groupMiddleware = [];

    private string $groupNamePrefix = '';

    public function get(string $path, array $handler, array $middleware = [], ?string $name = null): RouteDefinition
    {
        return $this->add('GET', $path, $handler, $middleware, $name);
    }

    public function post(string $path, array $handler, array $middleware = [], ?string $name = null): RouteDefinition
    {
        return $this->add('POST', $path, $handler, $middleware, $name);
    }

    public function put(string $path, array $handler, array $middleware = [], ?string $name = null): RouteDefinition
    {
        return $this->add('PUT', $path, $handler, $middleware, $name);
    }

    public function patch(string $path, array $handler, array $middleware = [], ?string $name = null): RouteDefinition
    {
        return $this->add('PATCH', $path, $handler, $middleware, $name);
    }

    public function delete(string $path, array $handler, array $middleware = [], ?string $name = null): RouteDefinition
    {
        return $this->add('DELETE', $path, $handler, $middleware, $name);
    }

    public function routes(): array
    {
        $list = [];

        foreach ($this->routes as $method => $routes) {
            foreach ($routes as $route) {
                $handler = $route['handler'] ?? [];

                $list[] = [
                    'method' => $method,
                    'path' => $route['path'] ?? '',
                    'name' => $route['name'] ?? '',
                    'controller' => (string)($handler[0] ?? ''),
                    'action' => (string)($handler[1] ?? ''),
                    'middleware' => $route['middleware'] ?? [],
                ];
            }
        }

        return $list;
    }

    public function group(array $options, callable $callback): void
    {
        $oldPrefix = $this->groupPrefix;
        $oldMiddleware = $this->groupMiddleware;
        $oldNamePrefix = $this->groupNamePrefix;

        $prefix = (string)($options['prefix'] ?? '');
        $namePrefix = (string)($options['as'] ?? $options['name'] ?? '');
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

        $this->groupMiddleware = array_values(array_unique(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        ))));

        $callback($this);

        $this->groupPrefix = $oldPrefix;
        $this->groupMiddleware = $oldMiddleware;
        $this->groupNamePrefix = $oldNamePrefix;
    }

    private function add(string $method, string $path, array $handler, array $middleware = [], ?string $name = null): RouteDefinition
    {
        $method = strtoupper($method);

        $path = $this->joinPaths($this->groupPrefix, $path);
        $path = $this->normalizePath($path);

        $middleware = array_values(array_unique(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        ))));

        $finalName = $this->applyNamePrefix($name);

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

        $index = array_key_last($this->routes[$method]);

        if ($finalName !== null && $finalName !== '') {
            $this->namedRoutes[$finalName] = $this->routes[$method][$index];
        }

        return new RouteDefinition($this, $method, (int)$index);
    }

    public function setRouteName(string $method, int $index, string $name): void
    {
        $method = strtoupper($method);

        if (!isset($this->routes[$method][$index])) {
            return;
        }

        $finalName = $this->applyNamePrefix($name);

        $this->routes[$method][$index]['name'] = $finalName;

        if ($finalName !== null && $finalName !== '') {
            $this->namedRoutes[$finalName] = $this->routes[$method][$index];
        }
    }

    public function addRouteMiddleware(string $method, int $index, array|string $middleware): void
    {
        $method = strtoupper($method);

        if (!isset($this->routes[$method][$index])) {
            return;
        }

        if (!is_array($middleware)) {
            $middleware = [$middleware];
        }

        $this->routes[$method][$index]['middleware'] = array_values(array_unique(array_filter(array_merge(
            $this->routes[$method][$index]['middleware'] ?? [],
            $middleware
        ))));
    }

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

    private function applyNamePrefix(?string $name): ?string
    {
        if ($name === null || $name === '') {
            return null;
        }

        if ($this->groupNamePrefix !== '' && !str_starts_with($name, $this->groupNamePrefix)) {
            return $this->groupNamePrefix . $name;
        }

        return $name;
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

3. Создай /local/mvc/Support/RouteGroup.php

Создай папку:

/local/mvc/Support/

Файл:

/local/mvc/Support/RouteGroup.php

Код:

<?php

namespace Local\Mvc\Support;

use Local\Mvc\Support\Facades\Route;

/**
 * RouteGroup
 *
 * Нужен для Laravel-like синтаксиса:
 *
 * Route::prefix('/admin')
 *     ->middleware(['auth', 'admin'])
 *     ->name('admin.')
 *     ->group(function () {
 *         Route::get('/users', [Controller::class, 'index']);
 *     });
 */
class RouteGroup
{
    public function __construct(
        private array $options = []
    ) {}

    public function prefix(string $prefix): self
    {
        $new = clone $this;
        $new->options['prefix'] = $prefix;

        return $new;
    }

    public function middleware(array|string $middleware): self
    {
        $new = clone $this;

        if (!is_array($middleware)) {
            $middleware = [$middleware];
        }

        $current = $new->options['middleware'] ?? [];

        if (!is_array($current)) {
            $current = [$current];
        }

        $new->options['middleware'] = array_values(array_unique(array_filter(array_merge(
            $current,
            $middleware
        ))));

        return $new;
    }

    public function name(string $prefix): self
    {
        return $this->as($prefix);
    }

    public function as(string $prefix): self
    {
        $new = clone $this;
        $new->options['as'] = $prefix;

        return $new;
    }

    public function group(callable $callback): void
    {
        Route::router()->group($this->options, $callback);
    }
}


---

4. Создай /local/mvc/Support/Facades/Route.php

Создай папку:

/local/mvc/Support/Facades/

Файл:

/local/mvc/Support/Facades/Route.php

Код:

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\RouteDefinition;
use Local\Mvc\Core\Router;
use Local\Mvc\Support\RouteGroup;
use RuntimeException;

/**
 * Route
 *
 * Laravel-like facade для маршрутов.
 */
class Route
{
    private static ?Router $router = null;

    public static function setRouter(Router $router): void
    {
        self::$router = $router;
    }

    public static function router(): Router
    {
        if (!(self::$router instanceof Router)) {
            throw new RuntimeException('ROUTER_NOT_INITIALIZED');
        }

        return self::$router;
    }

    public static function get(string $path, array $handler): RouteDefinition
    {
        return self::router()->get($path, $handler);
    }

    public static function post(string $path, array $handler): RouteDefinition
    {
        return self::router()->post($path, $handler);
    }

    public static function put(string $path, array $handler): RouteDefinition
    {
        return self::router()->put($path, $handler);
    }

    public static function patch(string $path, array $handler): RouteDefinition
    {
        return self::router()->patch($path, $handler);
    }

    public static function delete(string $path, array $handler): RouteDefinition
    {
        return self::router()->delete($path, $handler);
    }

    public static function prefix(string $prefix): RouteGroup
    {
        return (new RouteGroup())->prefix($prefix);
    }

    public static function middleware(array|string $middleware): RouteGroup
    {
        return (new RouteGroup())->middleware($middleware);
    }

    public static function name(string $prefix): RouteGroup
    {
        return (new RouteGroup())->name($prefix);
    }

    public static function group(array $options, callable $callback): void
    {
        self::router()->group($options, $callback);
    }
}


---

5. Обнови /local/mvc/Core/App.php

Найди место, где создаётся router:

$router = new Router();
self::$router = $router;

Сразу после этого добавь:

\Local\Mvc\Support\Facades\Route::setRouter($router);

Должно быть так:

$router = new Router();
self::$router = $router;

\Local\Mvc\Support\Facades\Route::setRouter($router);

require $routesFile;

$router->dispatch($request);


---

6. Добавь Laravel-like тест в /local/mvc_demo/routes.php

Вверху файла добавь:

use Local\Mvc\Support\Facades\Route;

Потом рядом с публичными маршрутами добавь:

Route::get('/laravel-like-ping', [HomeController::class, 'ping'])
    ->name('laravel.ping');

И добавь пример группы:

Route::prefix('/laravel-admin')
    ->middleware(['auth', 'admin'])
    ->name('laravel.admin.')
    ->group(function () {
        Route::get('/dashboard', [\Local\MvcDemo\Controllers\AdminController::class, 'dashboard'])
            ->name('dashboard');
    });

После этого будут работать маршруты:

/local/mvc_demo/laravel-like-ping
/local/mvc_demo/laravel-admin/dashboard

И в debug/routes появятся имена:

laravel.ping
laravel.admin.dashboard


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/laravel-like-ping

Потом:

https://bitrix24-stage.gaz.ru/local/mvc_demo/laravel-admin/dashboard

И затем:

https://bitrix24-stage.gaz.ru/local/mvc_demo/debug/routes


---

Что мы сделали

Теперь у нас есть Laravel-like маршруты:

Route::get('/path', [Controller::class, 'method'])
    ->middleware(['auth'])
    ->name('route.name');

И группы:

Route::prefix('/admin')
    ->middleware(['auth', 'admin'])
    ->name('admin.')
    ->group(function () {
        Route::get('/dashboard', [AdminController::class, 'dashboard'])
            ->name('dashboard');
    });

Простыми словами:

Router — двигатель маршрутизации.
Route facade — красивый Laravel-похожий интерфейс.

Дальше будем постепенно приводить фреймворк к Laravel-стилю:

Route facade
Request object
Controller
Middleware
Service container
Config
Logger
Validation
Redirect
Response
Models
Services

Следующий очень Laravel-like шаг — сделать service container: чтобы можно было не писать new UserService(), а фреймворк сам создавал нужные классы.