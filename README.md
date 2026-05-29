Да, держим курс именно на Laravel-like.

То есть дальше стараемся, чтобы было похоже на Laravel по стилю:

Route::get('/users', [AdminController::class, 'users'])
    ->middleware(['auth', 'admin'])
    ->name('admin.users.index');

И чтобы в контроллере можно было не писать:

$userService = new UserService();

А писать почти как в Laravel:

public function users(UserService $userService): Response
{
    ...
}

Фреймворк сам создаст UserService и передаст его в метод.

Это называется Service Container.


---

Что делаем сейчас

Добавляем контейнер:

/local/mvc/Core/Container.php

Он будет уметь:

создавать классы
подставлять зависимости
передавать Request
передавать Service в методы контроллера


---

1. Создай /local/mvc/Core/Container.php

<?php

namespace Local\Mvc\Core;

use ReflectionClass;
use ReflectionFunction;
use ReflectionMethod;
use ReflectionNamedType;
use RuntimeException;

/**
 * Container
 *
 * Laravel-like service container.
 *
 * Простыми словами:
 * это коробка, которая умеет сама создавать классы
 * и подставлять им нужные зависимости.
 */
class Container
{
    private array $bindings = [];

    private array $instances = [];

    /**
     * Зарегистрировать готовый объект.
     *
     * Например:
     * Request::class => $request
     */
    public function instance(string $abstract, object $instance): void
    {
        $this->instances[$abstract] = $instance;
    }

    /**
     * Зарегистрировать связь.
     *
     * Например:
     * LoggerInterface::class => FileLogger::class
     */
    public function bind(string $abstract, string|callable $concrete): void
    {
        $this->bindings[$abstract] = $concrete;
    }

    /**
     * Получить объект.
     */
    public function make(string $class, array $parameters = []): object
    {
        if (isset($this->instances[$class])) {
            return $this->instances[$class];
        }

        if (isset($this->bindings[$class])) {
            $concrete = $this->bindings[$class];

            if (is_callable($concrete)) {
                return $concrete($this);
            }

            $class = $concrete;
        }

        if (!class_exists($class)) {
            throw new RuntimeException('CONTAINER_CLASS_NOT_FOUND: ' . $class);
        }

        $reflection = new ReflectionClass($class);

        if (!$reflection->isInstantiable()) {
            throw new RuntimeException('CONTAINER_CLASS_NOT_INSTANTIABLE: ' . $class);
        }

        $constructor = $reflection->getConstructor();

        if ($constructor === null) {
            return new $class();
        }

        $dependencies = $this->resolveParameters($constructor, $parameters);

        return $reflection->newInstanceArgs($dependencies);
    }

    /**
     * Вызвать метод и автоматически подставить зависимости.
     *
     * Например:
     * public function users(UserService $service)
     *
     * Container сам создаст UserService.
     */
    public function call(array $callable, array $parameters = []): mixed
    {
        [$object, $method] = $callable;

        $reflection = new ReflectionMethod($object, $method);

        $dependencies = $this->resolveParameters($reflection, $parameters);

        return $reflection->invokeArgs($object, $dependencies);
    }

    private function resolveParameters(ReflectionMethod|\ReflectionFunctionAbstract $reflection, array $parameters = []): array
    {
        $dependencies = [];

        foreach ($reflection->getParameters() as $parameter) {
            $name = $parameter->getName();
            $type = $parameter->getType();

            /**
             * 1. Если есть параметр маршрута с таким именем — используем его.
             *
             * Например маршрут:
             * /users/{id}
             *
             * Метод:
             * userDetail(string $id)
             */
            if (array_key_exists($name, $parameters)) {
                $dependencies[] = $parameters[$name];
                continue;
            }

            /**
             * 2. Если параметр — класс, создаём его через контейнер.
             *
             * Например:
             * UserService $userService
             */
            if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
                $dependencies[] = $this->make($type->getName());
                continue;
            }

            /**
             * 3. Если есть значение по умолчанию — используем его.
             */
            if ($parameter->isDefaultValueAvailable()) {
                $dependencies[] = $parameter->getDefaultValue();
                continue;
            }

            throw new RuntimeException(
                'CONTAINER_CANNOT_RESOLVE_PARAMETER: $' . $name . ' in ' . $reflection->getName()
            );
        }

        return $dependencies;
    }
}


---

2. Обнови /local/mvc/Core/App.php

Нужно, чтобы App создал контейнер и положил туда Request и Router.

Замени файл полностью:

<?php

namespace Local\Mvc\Core;

/**
 * App
 *
 * Запускатель MVC-приложения.
 */
class App
{
    private static ?Router $router = null;

    private static ?Container $container = null;

    public static function run(?string $routesFile = null): void
    {
        $projectRoot = self::projectRoot();

        self::loadConfig($projectRoot);

        if ($routesFile === null) {
            $routesFile = $projectRoot . '/routes.php';
        }

        $request = Request::createFromGlobals();

        ErrorHandler::register($request);

        try {
            if (!is_file($routesFile)) {
                Response::html(
                    '<h1>500</h1><p>Файл маршрутов не найден.</p><pre>'
                    . htmlspecialchars($routesFile)
                    . '</pre>',
                    500
                )->send();

                return;
            }

            $container = new Container();
            self::$container = $container;

            $router = new Router();
            self::$router = $router;

            /**
             * Кладём важные объекты в контейнер.
             *
             * Теперь если какому-то классу нужен Request,
             * контейнер отдаст текущий Request.
             */
            $container->instance(Request::class, $request);
            $container->instance(Router::class, $router);
            $container->instance(Container::class, $container);

            \Local\Mvc\Support\Facades\Route::setRouter($router);

            require $routesFile;

            $router->dispatch($request);
        } catch (\Throwable $e) {
            ErrorHandler::renderThrowable($e);
        }
    }

    public static function router(): ?Router
    {
        return self::$router;
    }

    public static function container(): Container
    {
        if (!(self::$container instanceof Container)) {
            self::$container = new Container();
        }

        return self::$container;
    }

    public static function make(string $class, array $parameters = []): object
    {
        return self::container()->make($class, $parameters);
    }

    public static function route(string $name, array $params = [], array $query = []): string
    {
        if (!(self::$router instanceof Router)) {
            return '#router-not-ready';
        }

        return self::$router->url($name, $params, $query);
    }

    private static function loadConfig(string $projectRoot): void
    {
        $configFile = rtrim($projectRoot, '/') . '/config.php';

        $config = [];

        if (is_file($configFile)) {
            $loaded = require $configFile;

            if (is_array($loaded)) {
                $config = $loaded;
            }
        }

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

        Config::load($config);
    }

    public static function projectRoot(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_ROOT')) {
            return $_SERVER['DOCUMENT_ROOT'] . '/local/mvc';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/');
    }

    public static function projectUrl(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_URL')) {
            return '/local/mvc';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_URL, '/');
    }

    public static function projectNamespace(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_NAMESPACE')) {
            return 'Local\\Mvc\\';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_NAMESPACE, '\\') . '\\';
    }
}


---

3. Обнови dispatch() в /local/mvc/Core/Router.php

В файле:

/local/mvc/Core/Router.php

найди кусок:

$controller = new $controllerClass($request);

Замени на:

$controller = App::container()->make($controllerClass);

Потом найди:

$result = $controller->{$controllerMethod}(...array_values($routeParams));

Замени на:

$result = App::container()->call([$controller, $controllerMethod], $routeParams);

Итоговый важный кусок в dispatch() должен быть таким:

$controller = App::container()->make($controllerClass);

if (!$controllerMethod || !method_exists($controller, $controllerMethod)) {
    $methods = get_class_methods($controller);

    $this->serverError(
        'Метод контроллера не найден: ' . $controllerClass . '::' . (string)$controllerMethod
        . "\n\nPHP видит такие методы:\n"
        . implode("\n", $methods)
    );

    return;
}

$result = App::container()->call([$controller, $controllerMethod], $routeParams);

if ($result instanceof Response) {
    $result->send();
    return;
}


---

4. Проверяем, что Controller принимает Request

В /local/mvc/Core/Controller.php должно быть так:

public function __construct(?Request $request = null)
{
    $this->request = $request ?? Request::createFromGlobals();
}

Оставь как есть. Контейнер сам подставит текущий Request.


---

5. Теперь перепишем AdminController в Laravel-like стиле

Открой:

/local/mvc_demo/Controllers/AdminController.php

Замени файл полностью:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\Auth;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Services\UserService;

class AdminController extends Controller
{
    public function dashboard(UserService $userService): Response
    {
        return $this->render('admin/dashboard', [
            'title' => 'Админ-панель',
            'message' => 'Это защищённая админская страница. Сюда может зайти только администратор.',
            'user' => [
                'id' => Auth::id(),
                'login' => Auth::login(),
                'name' => Auth::name(),
                'email' => Auth::email(),
            ],
            'stats' => $userService->dashboardStats(),
        ]);
    }

    public function users(UserService $userService): Response
    {
        $page = (int)$this->request->get('page', 1);
        $search = trim((string)$this->request->get('q', ''));

        $result = $userService->paginateForTable($page, 10, $search);

        return $this->render('admin/users', [
            'title' => 'Пользователи',
            'users' => $result['items'],
            'pagination' => $result['pagination'],
            'search' => $result['search'],
        ]);
    }

    public function userDetail(string $id, UserService $userService): Response
    {
        $user = $userService->findForDetail((int)$id);

        if (!$user) {
            return Response::html(
                '<h1>404</h1><p>Пользователь не найден.</p>',
                404
            );
        }

        return $this->render('admin/user_detail', [
            'title' => 'Карточка пользователя',
            'user' => $user,
        ]);
    }
}

Обрати внимание:

public function users(UserService $userService): Response

Мы больше не пишем:

$userService = new UserService();

Это делает контейнер.


---

6. Перепишем UserApiController

Файл:

/local/mvc_demo/Controllers/UserApiController.php

Замени полностью:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\ApiController;
use Local\Mvc\Core\Response;
use Local\MvcDemo\Services\UserService;

class UserApiController extends ApiController
{
    public function index(UserService $userService): Response
    {
        $page = (int)$this->request->get('page', 1);
        $search = trim((string)$this->request->get('q', ''));

        $result = $userService->paginateForTable($page, 10, $search);

        return $this->ok([
            'items' => $result['items'],
            'pagination' => $result['pagination'],
            'search' => $result['search'],
        ]);
    }

    public function show(string $id, UserService $userService): Response
    {
        $user = $userService->findForDetail((int)$id);

        if (!$user) {
            $this->abortNotFound('Пользователь не найден', [
                'id' => (int)$id,
            ]);
        }

        return $this->ok([
            'user' => $user,
        ]);
    }
}


---

7. Добавим тест контейнера

В HomeController добавь метод:

public function containerTest(UserService $userService): Response
{
    return $this->success([
        'message' => 'Container работает. UserService был создан автоматически.',
        'stats' => $userService->dashboardStats(),
    ]);
}

И сверху добавь:

use Local\MvcDemo\Services\UserService;

В /local/mvc_demo/routes.php добавь маршрут:

Route::get('/container-test', [HomeController::class, 'containerTest'])
    ->name('container.test');


---

8. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/container-test

Должен быть JSON:

{
  "ok": true,
  "data": {
    "message": "Container работает. UserService был создан автоматически.",
    "stats": {
      "users_count": 123
    }
  }
}

Проверь ещё:

/local/mvc_demo/admin/dashboard
/local/mvc_demo/admin/users
/local/mvc_demo/api/users


---

Что мы сделали

Раньше было так:

public function users(): Response
{
    $userService = new UserService();
}

Теперь Laravel-like:

public function users(UserService $userService): Response
{
    ...
}

Фреймворк сам:

видит, что методу нужен UserService
создаёт UserService
передаёт его в метод

Это очень похоже на Laravel.

Главная мысль:

Container — это мозг, который создаёт зависимости.
Controller — просто говорит, что ему нужно.

Дальше можно сделать ещё ближе к Laravel: FormRequest — отдельные классы для валидации запросов, например StoreMessageRequest, чтобы в контроллере не писать Validator вручную.