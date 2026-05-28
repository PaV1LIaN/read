Отлично. Дальше делаем именованные маршруты.

Зачем это нужно

Сейчас во views мы руками пишем ссылки:

<a href="/local/mvc_demo/admin/users/<?= (int)$user['id'] ?>">

Это плохо, потому что если адрес изменится, например:

/admin/users/{id}

на:

/admin/people/{id}

придётся искать ссылки по всему проекту.

А мы хотим так:

<a href="<?= App::route('admin.users.show', ['id' => $user['id']]) ?>">

То есть ссылка строится не по строке URL, а по имени маршрута.


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
 */
class Router
{
    private array $routes = [];

    /**
     * Маршруты по имени.
     *
     * Например:
     * admin.users.show => /admin/users/{id:\d+}
     */
    private array $namedRoutes = [];

    private string $groupPrefix = '';

    private array $groupMiddleware = [];

    public function get(string $path, array $handler, array $middleware = [], ?string $name = null): void
    {
        $this->add('GET', $path, $handler, $middleware, $name);
    }

    public function post(string $path, array $handler, array $middleware = [], ?string $name = null): void
    {
        $this->add('POST', $path, $handler, $middleware, $name);
    }

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

    private function add(string $method, string $path, array $handler, array $middleware = [], ?string $name = null): void
    {
        $method = strtoupper($method);

        $path = $this->joinPaths($this->groupPrefix, $path);
        $path = $this->normalizePath($path);

        $middleware = array_values(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        )));

        $compiled = $this->compilePath($path);

        $route = [
            'method' => $method,
            'path' => $path,
            'pattern' => $compiled['pattern'],
            'params' => $compiled['params'],
            'handler' => $handler,
            'middleware' => $middleware,
            'name' => $name,
        ];

        $this->routes[$method][] = $route;

        if ($name !== null && $name !== '') {
            $this->namedRoutes[$name] = $route;
        }
    }

    /**
     * Собрать URL по имени маршрута.
     *
     * Пример:
     * $router->url('admin.users.show', ['id' => 5])
     *
     * Вернёт:
     * /local/mvc_demo/admin/users/5
     */
    public function url(string $name, array $params = [], array $query = []): string
    {
        if (!isset($this->namedRoutes[$name])) {
            return '#route-not-found-' . rawurlencode($name);
        }

        $route = $this->namedRoutes[$name];

        $path = (string)($route['path'] ?? '/');

        /**
         * Заменяем параметры маршрута.
         *
         * Было:
         * /admin/users/{id:\d+}
         *
         * Стало:
         * /admin/users/5
         */
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

2. Обновляем /local/mvc/Core/App.php

Добавим короткий помощник:

App::route(...)

Открой:

/local/mvc/Core/App.php

Внутрь класса добавь метод:

/**
 * Собрать URL по имени маршрута.
 */
public static function route(string $name, array $params = [], array $query = []): string
{
    if (!(self::$router instanceof Router)) {
        return '#router-not-ready';
    }

    return self::$router->url($name, $params, $query);
}

Лучше вставить сразу после метода:

public static function router(): ?Router


---

3. Обновляем /local/mvc_demo/routes.php

Теперь дадим имена нескольким маршрутам.

Найди эти строки:

$router->get('/', [HomeController::class, 'index']);
$router->get('/about', [HomeController::class, 'about']);
$router->get('/form', [FormController::class, 'index']);

Замени на:

$router->get('/', [HomeController::class, 'index'], [], 'home');

$router->get('/about', [HomeController::class, 'about'], [], 'about');

$router->get('/form', [FormController::class, 'index'], [], 'form.index');

В админской группе замени:

$router->get('/dashboard', [AdminController::class, 'dashboard']);
$router->get('/users', [AdminController::class, 'users']);
$router->get('/users/{id:\d+}', [AdminController::class, 'userDetail']);

на:

$router->get('/dashboard', [AdminController::class, 'dashboard'], [], 'admin.dashboard');

$router->get('/users', [AdminController::class, 'users'], [], 'admin.users.index');

$router->get('/users/{id:\d+}', [AdminController::class, 'userDetail'], [], 'admin.users.show');


---

4. Обновляем debug-страницу маршрутов

Файл:

/local/mvc_demo/Views/debug/routes.php

В таблице добавим колонку Имя.

Найди заголовки таблицы и сделай так:

<tr>
    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Метод</th>
    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Путь</th>
    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Имя</th>
    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Контроллер</th>
    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Action</th>
    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Middleware</th>
</tr>

И в строке маршрута после колонки Путь добавь:

<td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
    <span class="mvc-code">
        <?= htmlspecialcharsbx($route['name'] ?? '') ?>
    </span>
</td>


---

5. Обновляем /local/mvc_demo/Views/admin/users.php

Сейчас ссылка такая:

<a href="/local/mvc_demo/admin/users/<?= (int)($user['id'] ?? 0) ?>">
    Открыть
</a>

Замени её на:

<a href="<?= htmlspecialcharsbx(\Local\Mvc\Core\App::route('admin.users.show', [
    'id' => (int)($user['id'] ?? 0),
])) ?>">
    Открыть
</a>

Теперь view не знает точный URL. Она знает только имя маршрута:

admin.users.show


---

6. Обновляем /local/mvc_demo/Views/admin/user_detail.php

Ссылка назад сейчас такая:

<a href="/local/mvc_demo/admin/users">
    ← Назад к списку
</a>

Замени на:

<a href="<?= htmlspecialcharsbx(\Local\Mvc\Core\App::route('admin.users.index')) ?>">
    ← Назад к списку
</a>


---

7. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь список маршрутов:

/local/mvc_demo/debug/routes

Теперь должна появиться колонка с именами:

home
about
form.index
admin.dashboard
admin.users.index
admin.users.show

Проверь пользователей:

/local/mvc_demo/admin/users

Нажми “Открыть”. Должно открыть карточку пользователя.


---

Что мы сделали

Раньше ссылка была привязана к адресу:

/local/mvc_demo/admin/users/5

Теперь ссылка привязана к имени:

App::route('admin.users.show', ['id' => 5])

Главная мысль:

URL может поменяться.
Имя маршрута остаётся стабильным.

Для будущего sitebuilder это будет очень удобно:

App::route('sites.edit', ['siteId' => 10])
App::route('pages.edit', ['siteId' => 10, 'pageId' => 55])
App::route('blocks.store', ['pageId' => 55])

Следующий шаг — сделать redirectRoute(), чтобы в контроллере тоже не писать URL руками.