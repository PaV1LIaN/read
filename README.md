Идём дальше. Следующий важный кирпич — динамические маршруты.

Сейчас у нас маршруты только точные:

$router->get('/about', [HomeController::class, 'about']);
$router->get('/admin/users', [AdminController::class, 'users']);

Но в реальном проекте нам нужны адреса с ID:

/local/mvc_demo/admin/users/5
/local/sitebuilder/sites/10/edit
/local/glab/applications/25

То есть маршрут должен уметь понимать:

$router->get('/admin/users/{id}', [AdminController::class, 'userDetail']);

И если пользователь открыл:

/admin/users/5

контроллер должен получить:

$id = 5;


---

1. Обновляем Request.php

Открой файл:

/local/mvc/Core/Request.php

Внутрь класса добавь свойство:

private array $routeParams = [];

Лучше вставить рядом с остальными свойствами:

private array $get;
private array $post;
private array $server;
private array $files;
private array $routeParams = [];

И в конец класса, перед последней }, добавь методы:

/**
 * Сохранить параметры маршрута.
 *
 * Например:
 * /users/{id}
 * /users/5
 *
 * станет:
 * ['id' => '5']
 */
public function setRouteParams(array $params): void
{
    $this->routeParams = $params;
}

/**
 * Получить параметр маршрута.
 *
 * Например:
 * $request->route('id')
 */
public function route(string $key, mixed $default = null): mixed
{
    return $this->routeParams[$key] ?? $default;
}

/**
 * Получить все параметры маршрута.
 */
public function routeParams(): array
{
    return $this->routeParams;
}


---

2. Полностью замени /local/mvc/Core/Router.php

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
 * - динамические параметры вида /users/{id}
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

        /**
         * Кладём параметры маршрута в Request.
         *
         * Например:
         * /users/{id}
         * /users/5
         *
         * $request->route('id') вернёт 5.
         */
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

        /**
         * Передаём параметры маршрута прямо в метод контроллера.
         *
         * Маршрут:
         * /users/{id}
         *
         * Метод:
         * userDetail($id)
         */
        $result = $controller->{$controllerMethod}(...array_values($routeParams));

        if ($result instanceof Response) {
            $result->send();
            return;
        }
    }

    /**
     * Найти подходящий маршрут.
     */
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
     * Превратить путь маршрута в регулярку.
     *
     * Было:
     * /admin/users/{id}
     *
     * Стало:
     * #^/admin/users/(?P<id>[^/]+)$#u
     */
    private function compilePath(string $path): array
    {
        $params = [];

        $pattern = preg_replace_callback('#\{([a-zA-Z_][a-zA-Z0-9_]*)\}#', function ($matches) use (&$params) {
            $name = $matches[1];
            $params[] = $name;

            return '(?P<' . $name . '>[^/]+)';
        }, $path);

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

3. Добавляем метод в AdminController

Открой:

/local/mvc_demo/Controllers/AdminController.php

Добавь внутрь класса метод:

public function userDetail(string $id): Response
{
    return $this->render('admin/user_detail', [
        'title' => 'Карточка пользователя',
        'userId' => (int)$id,
    ]);
}


---

4. Создай view /local/mvc_demo/Views/admin/user_detail.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Карточка пользователя') ?>
    </h1>

    <p class="mvc-page-text">
        Это тестовая страница динамического маршрута.
    </p>

    <div class="mvc-info">
        <b>Что произошло:</b>

        <ol>
            <li>
                В routes.php есть маршрут:
                <span class="mvc-code">/admin/users/{id}</span>
            </li>

            <li>
                Ты открыл адрес с конкретным ID.
            </li>

            <li>
                Router вытащил ID из адреса.
            </li>

            <li>
                Контроллер получил ID:
                <span class="mvc-code"><?= htmlspecialcharsbx($userId ?? '') ?></span>
            </li>
        </ol>
    </div>
</div>


---

5. Обнови /local/mvc_demo/routes.php

Внутри админской группы добавь маршрут:

$router->get('/users/{id}', [AdminController::class, 'userDetail']);

Должно быть примерно так:

$router->group([
    'prefix' => '/admin',
    'middleware' => ['admin'],
], function (Router $router) {
    $router->get('/dashboard', [AdminController::class, 'dashboard']);
    $router->get('/users', [AdminController::class, 'users']);
    $router->get('/users/{id}', [AdminController::class, 'userDetail']);
});


---

6. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users/5

Должна открыться страница:

Карточка пользователя
Контроллер получил ID: 5

Потом проверь другой ID:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin/users/123

Должно показать:

Контроллер получил ID: 123


---

Что мы сделали

Раньше Router понимал только точные адреса:

/admin/users
/about
/form

Теперь он понимает адреса с параметрами:

/admin/users/{id}
/sites/{siteId}/pages/{pageId}
/applications/{id}

Пример для будущего sitebuilder:

$router->get('/sites/{siteId}/edit', [SiteController::class, 'edit']);
$router->get('/sites/{siteId}/pages/{pageId}', [PageController::class, 'edit']);

Контроллер:

public function edit(string $siteId): Response
{
    // $siteId пришёл из адреса
}

Главная мысль:

{id} — это дырка в маршруте.
Router достаёт значение из адреса.
Controller получает это значение.

Следующим шагом можно сделать ограничения параметров, чтобы {id} принимал только цифры, а не любой текст.