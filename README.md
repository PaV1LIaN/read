Ошибка говорит почти прямо:

syntax error, unexpected token "<", expecting "function" or "const"

Это значит, что в файле:

/local/mvc/Core/Router.php

на строке 36 PHP встретил символ <.

Скорее всего, ты случайно вставил внутрь класса повторный открывающий тег:

<?php

Например было так:

class Router
{
    public function get(...)
    {
        ...
    }

    <?php // ← вот это внутри класса ломает файл

Чтобы не искать вручную, проще заменить Router.php полностью.


---

Замени полностью /local/mvc/Core/Router.php

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
 * - список маршрутов для debug-страницы
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

    /**
     * Получить список всех маршрутов.
     *
     * Нужно для debug-страницы /debug/routes.
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
                    'controller' => (string)$controller,
                    'action' => (string)$action,
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

Потом сбрось OPcache

opcache_reset();
echo 'OPcache reset OK';


---

Проверь

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/debug/routes

Должна открыться таблица маршрутов.

Главное правило: в PHP-файле Router.php открывающий тег:

<?php

должен быть только один раз — в самом начале файла.