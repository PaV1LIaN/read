Идём дальше. Делаем группы маршрутов.

Сейчас у нас так:

$router->get('/me', [HomeController::class, 'me'], ['auth']);
$router->get('/admin-only', [HomeController::class, 'adminOnly'], ['auth', 'admin']);

Это работает. Но если будет 20 защищённых страниц, каждый раз писать ['auth'] неудобно.

Мы хотим так:

$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/me', [HomeController::class, 'me']);
    $router->get('/profile', [ProfileController::class, 'index']);
});

Простыми словами:

Всё, что внутри этой группы, доступно только авторизованным.


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
 * Он знает:
 * - какой URL открыл пользователь
 * - какой контроллер вызвать
 * - какие middleware применить
 */
class Router
{
    private array $routes = [];

    /**
     * Текущий префикс группы.
     *
     * Например:
     * /admin
     */
    private string $groupPrefix = '';

    /**
     * Текущие middleware группы.
     *
     * Например:
     * ['auth']
     */
    private array $groupMiddleware = [];

    /**
     * GET-маршрут.
     */
    public function get(string $path, array $handler, array $middleware = []): void
    {
        $this->add('GET', $path, $handler, $middleware);
    }

    /**
     * POST-маршрут.
     */
    public function post(string $path, array $handler, array $middleware = []): void
    {
        $this->add('POST', $path, $handler, $middleware);
    }

    /**
     * Группа маршрутов.
     *
     * Пример:
     *
     * $router->group(['middleware' => ['auth']], function (Router $router) {
     *     $router->get('/me', [HomeController::class, 'me']);
     * });
     *
     * Или с префиксом:
     *
     * $router->group(['prefix' => '/admin', 'middleware' => ['auth', 'admin']], function (Router $router) {
     *     $router->get('/dashboard', [AdminController::class, 'dashboard']);
     * });
     */
    public function group(array $options, callable $callback): void
    {
        /**
         * Запоминаем старые настройки.
         * Это нужно, чтобы после группы всё вернулось обратно.
         */
        $oldPrefix = $this->groupPrefix;
        $oldMiddleware = $this->groupMiddleware;

        $prefix = (string)($options['prefix'] ?? '');
        $middleware = $options['middleware'] ?? [];

        if (!is_array($middleware)) {
            $middleware = [$middleware];
        }

        /**
         * Добавляем префикс группы к старому префиксу.
         */
        if ($prefix !== '') {
            $this->groupPrefix = $this->joinPaths($this->groupPrefix, $prefix);
        }

        /**
         * Добавляем middleware группы к старым middleware.
         */
        $this->groupMiddleware = array_values(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        )));

        /**
         * Выполняем маршруты внутри группы.
         */
        $callback($this);

        /**
         * Возвращаем старые настройки.
         *
         * Иначе middleware группы случайно применятся
         * к следующим маршрутам.
         */
        $this->groupPrefix = $oldPrefix;
        $this->groupMiddleware = $oldMiddleware;
    }

    /**
     * Добавить маршрут.
     */
    private function add(string $method, string $path, array $handler, array $middleware = []): void
    {
        $method = strtoupper($method);

        /**
         * Если маршрут внутри группы с prefix,
         * склеиваем prefix + path.
         *
         * Например:
         * prefix = /admin
         * path = /dashboard
         *
         * получится:
         * /admin/dashboard
         */
        $path = $this->joinPaths($this->groupPrefix, $path);

        /**
         * Склеиваем middleware группы и middleware конкретного маршрута.
         */
        $middleware = array_values(array_filter(array_merge(
            $this->groupMiddleware,
            $middleware
        )));

        $path = $this->normalizePath($path);

        $this->routes[$method][$path] = [
            'handler' => $handler,
            'middleware' => $middleware,
        ];
    }

    /**
     * Запустить нужный контроллер по текущему запросу.
     */
    public function dispatch(Request $request): void
    {
        $method = $request->method();
        $path = $this->normalizePath($request->path());

        if (!isset($this->routes[$method][$path])) {
            $this->notFound($method, $path);
            return;
        }

        $route = $this->routes[$method][$path];

        $handler = $route['handler'] ?? [];
        $middlewares = $route['middleware'] ?? [];

        /**
         * Middleware выполняются ДО контроллера.
         */
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

        $result = $controller->{$controllerMethod}();

        if ($result instanceof Response) {
            $result->send();
            return;
        }
    }

    /**
     * Склеить два пути.
     *
     * Например:
     * /admin + /dashboard = /admin/dashboard
     */
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

    /**
     * Привести путь к нормальному виду.
     *
     * ''        => '/'
     * 'me'      => '/me'
     * '/me/'    => '/me'
     * '/admin/' => '/admin'
     */
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

    /**
     * 404 — маршрут не найден.
     */
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

    /**
     * 500 — ошибка внутри MVC.
     */
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

2. Обновляем /local/mvc_demo/routes.php

Теперь сделаем красиво.

Файл:

/local/mvc_demo/routes.php

замени на:

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;

/** @var Router $router */

/**
 * Публичные маршруты.
 *
 * Сюда можно без авторизации.
 */
$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);

/**
 * Группа только для авторизованных пользователей.
 *
 * Всё внутри автоматически получит middleware auth.
 */
$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/me', [HomeController::class, 'me']);

    /**
     * Вложенная группа только для администраторов.
     *
     * Здесь уже будет:
     * auth + admin
     */
    $router->group(['middleware' => ['admin']], function (Router $router) {
        $router->get('/admin-only', [HomeController::class, 'adminOnly']);
    });
});

Что здесь происходит

Вот это:

$router->group(['middleware' => ['auth']], function (Router $router) {

означает:

Все маршруты внутри требуют авторизацию.

А вот это:

$router->group(['middleware' => ['admin']], function (Router $router) {

внутри auth-группы означает:

Все маршруты внутри требуют авторизацию + права администратора.


---

3. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Публичный маршрут:

https://bitrix24-stage.gaz.ru/local/mvc_demo/ping

Должен работать без ограничений.

Авторизованный маршрут:

https://bitrix24-stage.gaz.ru/local/mvc_demo/me

Должен вернуть данные текущего пользователя.

Админский маршрут:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin-only

Если ты админ, вернёт успех.


---

Что мы сделали

Раньше было так:

$router->get('/me', [HomeController::class, 'me'], ['auth']);
$router->get('/admin-only', [HomeController::class, 'adminOnly'], ['auth', 'admin']);

Теперь так:

$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/me', [HomeController::class, 'me']);

    $router->group(['middleware' => ['admin']], function (Router $router) {
        $router->get('/admin-only', [HomeController::class, 'adminOnly']);
    });
});

Простыми словами:

Открыли комнату "только для авторизованных".
Внутри неё сделали ещё одну комнату "только для админов".


---

Зачем это нужно в реальных проектах

Например, в sitebuilder можно будет сделать так:

$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/', [DashboardController::class, 'index']);
    $router->get('/sites', [SiteController::class, 'index']);
    $router->get('/pages', [PageController::class, 'index']);

    $router->group(['middleware' => ['admin']], function (Router $router) {
        $router->get('/settings', [SettingsController::class, 'index']);
    });
});

И сразу понятно:

Весь sitebuilder требует входа.
А настройки доступны только админам.


---

Следующий хороший шаг — сделать префиксы групп, чтобы можно было удобно писать:

$router->group(['prefix' => '/admin', 'middleware' => ['auth', 'admin']], function (Router $router) {
    $router->get('/dashboard', [AdminController::class, 'dashboard']);
    $router->get('/users', [AdminController::class, 'users']);
});

И получить адреса:

/local/mvc_demo/admin/dashboard
/local/mvc_demo/admin/users