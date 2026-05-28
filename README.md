Идём дальше. Сейчас сделаем ограничения параметров маршрута.

Сейчас у нас маршрут:

$router->get('/users/{id}', [AdminController::class, 'userDetail']);

Он принимает всё:

/admin/users/5       — ок
/admin/users/123     — ок
/admin/users/test    — тоже ок, хотя нам это не нужно

А мы хотим сказать:

id должен быть только числом

Чтобы было так:

$router->get('/users/{id:\d+}', [AdminController::class, 'userDetail']);


---

1. Заменяем /local/mvc/Core/Router.php

Полностью замени файл:

/local/mvc/Core/Router.php

на этот:

<?php

namespace Local\Mvc\Core;

/**
 * Router
 *
 * Диспетчер маршрутов.
 *
 * Умеет:
 * - GET / POST маршруты
 * - middleware
 * - группы
 * - prefix
 * - динамические параметры /users/{id}
 * - ограничения параметров /users/{id:\d+}
 */
class Router
{
    private array $routes = [];

    private string $groupPrefix = '';

    private array $groupMiddleware = [];

    public function get(string $path, array $handler, array $middleware = []): void
    {
        $this->add('GET', $path, $handler, $middleware);
    }

    public function post(string $path, array $handler, array $middleware = []): void
    {
        $this->add('POST', $path, $handler, $middleware);
    }

    public function group(array $options, callable $callback): void
    {
        $oldPrefix = $this->groupPrefix;
        $oldMiddleware = $this->groupMiddleware;

        $prefix = (string)($options['prefix'] ?? '');
        $middleware = $options['middleware'] ?? [];

        if (!is_array($middleware)) {
            $middleware = [$middleware];
        }

        if ($prefix !== '') {
            $this->groupPrefix = $this->joinPaths($this->groupPrefix, $prefix);
        }

        $this->groupMiddleware = array_values(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        )));

        $callback($this);

        $this->groupPrefix = $oldPrefix;
        $this->groupMiddleware = $oldMiddleware;
    }

    private function add(string $method, string $path, array $handler, array $middleware = []): void
    {
        $method = strtoupper($method);

        $path = $this->joinPaths($this->groupPrefix, $path);
        $path = $this->normalizePath($path);

        $middleware = array_values(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        )));

        $compiled = $this->compilePath($path);

        $this->routes[$method][] = [
            'path' => $path,
            'pattern' => $compiled['pattern'],
            'params' => $compiled['params'],
            'handler' => $handler,
            'middleware' => $middleware,
        ];
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

    /**
     * Превратить маршрут в регулярку.
     *
     * Пример 1:
     * /admin/users/{id}
     *
     * станет:
     * #^/admin/users/(?P<id>[^/]+)$#u
     *
     * Пример 2:
     * /admin/users/{id:\d+}
     *
     * станет:
     * #^/admin/users/(?P<id>\d+)$#u
     *
     * То есть:
     * {id}       — любой текст до /
     * {id:\d+}  — только цифры
     * {slug:[a-z0-9-]+} — только slug
     */
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

            /**
             * Кусок обычного текста до параметра.
             */
            $staticPart = substr($path, $offset, $position - $offset);
            $pattern .= preg_quote($staticPart, '#');

            $name = $matches[1][$index][0];

            /**
             * Если ограничение не указано,
             * параметр принимает любой текст кроме "/".
             */
            $rule = $matches[2][$index][0] ?? '[^/]+';

            /**
             * На всякий случай экранируем разделитель регулярки.
             */
            $rule = str_replace('#', '\#', $rule);

            $params[] = $name;

            $pattern .= '(?P<' . $name . '>' . $rule . ')';

            $offset = $position + strlen($full);
        }

        /**
         * Остаток обычного текста после последнего параметра.
         */
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

2. Обнови маршрут в /local/mvc_demo/routes.php

Найди:

$router->get('/users/{id}', [AdminController::class, 'userDetail']);

Замени на:

$router->get('/users/{id:\d+}', [AdminController::class, 'userDetail']);

Полностью админская группа должна быть примерно такая:

$router->group([
    'prefix' => '/admin',
    'middleware' => ['admin'],
], function (Router $router) {
    $router->get('/dashboard', [AdminController::class, 'dashboard']);
    $router->get('/users', [AdminController::class, 'users']);
    $router->get('/users/{id:\d+}', [AdminController::class, 'userDetail']);
});


---

3. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users/5

Должно работать.

Теперь открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users/test

Должно быть:

404
Маршрут не найден

Это правильно, потому что test — не число.


---

Что мы сделали

Раньше:

/users/{id}

означало:

id может быть чем угодно

Теперь:

/users/{id:\d+}

означает:

id только цифры

Примеры на будущее

Для ID:

$router->get('/sites/{id:\d+}', [SiteController::class, 'show']);

Для slug:

$router->get('/pages/{slug:[a-z0-9-]+}', [PageController::class, 'show']);

Для кода:

$router->get('/items/{code:[A-Z0-9_]+}', [ItemController::class, 'show']);

Главная мысль:

{id} — любое значение
{id:\d+} — только число
{slug:[a-z0-9-]+} — только красивый текстовый код

Теперь Router стал намного ближе к настоящему фреймворку.