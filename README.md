Идём дальше.

Сейчас у нас проверка авторизации делается внутри метода контроллера:

if ($response = $this->requireAuth()) {
    return $response;
}

Это работает, но если таких методов будет много, придётся повторять одно и то же.

Мы хотим так:

$router->get('/me', [HomeController::class, 'me'], ['auth']);

Это будет означать:

Маршрут /me доступен только авторизованным.

А так:

$router->get('/admin-only', [HomeController::class, 'adminOnly'], ['auth', 'admin']);

Это будет означать:

Маршрут /admin-only доступен только авторизованным администраторам.


---

1. Создай /local/mvc/Core/Middleware.php

<?php

namespace Local\Mvc\Core;

/**
 * Middleware
 *
 * Это проверка, которая выполняется ДО контроллера.
 *
 * Простыми словами:
 * пользователь подошёл к двери,
 * middleware проверил пропуск,
 * и только потом пустил в контроллер.
 */
class Middleware
{
    /**
     * Запустить список middleware.
     *
     * Если всё хорошо — возвращаем null.
     * Если доступ запрещён — возвращаем Response с ошибкой.
     */
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

    /**
     * Обработать один middleware.
     */
    private static function handleOne(string $middleware, Request $request): ?Response
    {
        /**
         * auth — только авторизованные пользователи.
         */
        if ($middleware === 'auth') {
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

        /**
         * admin — только администраторы Битрикса.
         */
        if ($middleware === 'admin') {
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

        /**
         * Если middleware неизвестен — это ошибка разработчика.
         */
        return Response::json([
            'ok' => false,
            'error' => 'UNKNOWN_MIDDLEWARE',
            'details' => [
                'middleware' => $middleware,
            ],
        ], 500);
    }
}


---

2. Замени /local/mvc/Core/Router.php

Теперь Router должен хранить не только контроллер, но и middleware.

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
     * GET-маршрут.
     *
     * Пример:
     * $router->get('/me', [HomeController::class, 'me'], ['auth']);
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
     * Добавить маршрут.
     */
    private function add(string $method, string $path, array $handler, array $middleware = []): void
    {
        $method = strtoupper($method);
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
         * ВАЖНО:
         * middleware выполняется ДО контроллера.
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
     * Привести путь к нормальному виду.
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

3. Обнови /local/mvc_demo/routes.php

Теперь часть маршрутов сделаем публичными, а часть защищёнными.

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;

/** @var Router $router */

/**
 * Публичные маршруты.
 * Сюда можно без авторизации.
 */
$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);

/**
 * Только авторизованный пользователь.
 */
$router->get('/me', [HomeController::class, 'me'], ['auth']);

/**
 * Только администратор.
 */
$router->get('/admin-only', [HomeController::class, 'adminOnly'], ['auth', 'admin']);


---

4. Обнови /local/mvc_demo/Controllers/HomeController.php

Если метода me() уже есть, оставь его.
Добавь новый метод adminOnly() внутрь класса:

public function adminOnly(): Response
{
    return $this->success([
        'message' => 'Ты администратор, доступ разрешён.',
        'user_id' => \Local\Mvc\Core\Auth::id(),
        'login' => \Local\Mvc\Core\Auth::login(),
    ]);
}

Полный пример контроллера:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class HomeController extends Controller
{
    public function index(): Response
    {
        $name = (string)$this->request->get('name', 'Гость');

        return $this->render('home/index', [
            'title' => 'MVC Demo',
            'message' => 'Привет, ' . $name . '! Это отдельный проект, который использует общий фреймворк.',
        ]);
    }

    public function about(): Response
    {
        return $this->render('home/index', [
            'title' => 'О проекте MVC Demo',
            'message' => 'Этот проект лежит в /local/mvc_demo, а фреймворк лежит отдельно в /local/mvc.',
        ]);
    }

    public function ping(): Response
    {
        return $this->success([
            'message' => 'pong',
            'project' => 'mvc_demo',
            'framework' => 'local_mvc',
            'path' => $this->request->path(),
        ]);
    }

    public function me(): Response
    {
        return $this->success([
            'id' => Auth::id(),
            'login' => Auth::login(),
            'name' => Auth::name(),
            'email' => Auth::email(),
            'is_admin' => Auth::isAdmin(),
            'groups' => Auth::groups(),
        ]);
    }

    public function adminOnly(): Response
    {
        return $this->success([
            'message' => 'Ты администратор, доступ разрешён.',
            'id' => Auth::id(),
            'login' => Auth::login(),
        ]);
    }
}

Обрати внимание: теперь в me() уже не нужно писать:

if ($response = $this->requireAuth()) {
    return $response;
}

Потому что защиту делает маршрут:

$router->get('/me', [HomeController::class, 'me'], ['auth']);


---

5. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Публичный маршрут:

https://bitrix24-stage.gaz.ru/local/mvc_demo/ping

Должен работать для всех.

Только авторизованный:

https://bitrix24-stage.gaz.ru/local/mvc_demo/me

Если ты авторизован в Битриксе — вернёт данные пользователя.

Только администратор:

https://bitrix24-stage.gaz.ru/local/mvc_demo/admin-only

Если ты админ — вернёт успех.
Если не админ — вернёт:

{
  "ok": false,
  "error": "ADMIN_REQUIRED",
  "details": {
    "message": "Нужны права администратора"
  }
}


---

Что мы сделали

Раньше защита была внутри контроллера:

public function me(): Response
{
    if ($response = $this->requireAuth()) {
        return $response;
    }

    return $this->success([...]);
}

Теперь защита находится на уровне маршрута:

$router->get('/me', [HomeController::class, 'me'], ['auth']);

Это лучше, потому что сразу видно:

/me          — только авторизованные
/admin-only  — только администраторы
/ping        — публичный

Главная мысль

Middleware — это охранник до контроллера.

Браузер
  ↓
Router
  ↓
Middleware проверяет доступ
  ↓
Controller запускается только если доступ разрешён

Следующий шаг — сделать группы маршрутов, чтобы не писать ['auth'] у каждого маршрута вручную. Например:

$router->group(['middleware' => ['auth']], function (Router $router) {
    $router->get('/me', [HomeController::class, 'me']);
    $router->get('/profile', [ProfileController::class, 'index']);
});