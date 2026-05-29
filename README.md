Отлично. Дальше делаем ещё один Laravel-like кирпич — Service Providers.

В Laravel есть файлы-поставщики сервисов, например:

AppServiceProvider
RouteServiceProvider
AuthServiceProvider

Простыми словами:

ServiceProvider — это место, где проект говорит фреймворку:
"Вот мои сервисы, вот как их создавать, вот что надо настроить при запуске".


---

1. Обнови /local/mvc/Core/Container.php

Полностью замени файл:

<?php

namespace Local\Mvc\Core;

use ReflectionClass;
use ReflectionFunctionAbstract;
use ReflectionMethod;
use ReflectionNamedType;
use RuntimeException;

/**
 * Container
 *
 * Laravel-like service container.
 */
class Container
{
    private array $bindings = [];

    private array $instances = [];

    private array $singletons = [];

    public function instance(string $abstract, object $instance): void
    {
        $this->instances[$abstract] = $instance;
    }

    public function bind(string $abstract, string|callable $concrete): void
    {
        $this->bindings[$abstract] = $concrete;
    }

    public function singleton(string $abstract, string|callable $concrete): void
    {
        $this->bindings[$abstract] = $concrete;
        $this->singletons[$abstract] = true;
    }

    public function make(string $class, array $parameters = []): object
    {
        if (isset($this->instances[$class])) {
            return $this->instances[$class];
        }

        $abstract = $class;

        if (isset($this->bindings[$class])) {
            $concrete = $this->bindings[$class];

            if (is_callable($concrete)) {
                $object = $concrete($this);

                if (!is_object($object)) {
                    throw new RuntimeException('CONTAINER_BINDING_DID_NOT_RETURN_OBJECT: ' . $class);
                }

                if (!empty($this->singletons[$abstract])) {
                    $this->instances[$abstract] = $object;
                }

                return $object;
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
            $object = new $class();

            if (!empty($this->singletons[$abstract])) {
                $this->instances[$abstract] = $object;
            }

            return $object;
        }

        $dependencies = $this->resolveParameters($constructor, $parameters);

        $object = $reflection->newInstanceArgs($dependencies);

        if (!empty($this->singletons[$abstract])) {
            $this->instances[$abstract] = $object;
        }

        return $object;
    }

    public function call(array $callable, array $parameters = []): mixed
    {
        [$object, $method] = $callable;

        $reflection = new ReflectionMethod($object, $method);

        $dependencies = $this->resolveParameters($reflection, $parameters);

        return $reflection->invokeArgs($object, $dependencies);
    }

    private function resolveParameters(ReflectionMethod|ReflectionFunctionAbstract $reflection, array $parameters = []): array
    {
        $dependencies = [];

        foreach ($reflection->getParameters() as $parameter) {
            $name = $parameter->getName();
            $type = $parameter->getType();

            if (array_key_exists($name, $parameters)) {
                $dependencies[] = $parameters[$name];
                continue;
            }

            if ($type instanceof ReflectionNamedType && !$type->isBuiltin()) {
                $object = $this->make($type->getName());

                if ($object instanceof FormRequest) {
                    $object->validateResolved();
                }

                $dependencies[] = $object;
                continue;
            }

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

2. Создай /local/mvc/Core/ServiceProvider.php

<?php

namespace Local\Mvc\Core;

/**
 * ServiceProvider
 *
 * Laravel-like поставщик сервисов.
 *
 * register() — регистрируем сервисы в контейнере.
 * boot()     — действия после регистрации.
 */
abstract class ServiceProvider
{
    public function __construct(
        protected Container $app
    ) {}

    public function register(): void
    {
        //
    }

    public function boot(): void
    {
        //
    }
}


---

3. Обнови /local/mvc/Core/App.php

Полностью замени файл:

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

            $container->instance(Request::class, $request);
            $container->instance(Router::class, $router);
            $container->instance(Container::class, $container);

            /**
             * Регистрируем service providers проекта.
             */
            self::registerProviders($container);

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

            'providers' => [],

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

        Config::load($config);
    }

    private static function registerProviders(Container $container): void
    {
        $providerClasses = Config::get('providers', []);

        if (!is_array($providerClasses)) {
            return;
        }

        $providers = [];

        foreach ($providerClasses as $providerClass) {
            if (!is_string($providerClass) || $providerClass === '') {
                continue;
            }

            $provider = new $providerClass($container);

            if (!$provider instanceof ServiceProvider) {
                throw new \RuntimeException('SERVICE_PROVIDER_INVALID: ' . $providerClass);
            }

            $provider->register();

            $providers[] = $provider;
        }

        foreach ($providers as $provider) {
            $provider->boot();
        }
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

4. Добавь helper app() в /local/mvc/helpers.php

В конец файла добавь:

if (!function_exists('app')) {
    /**
     * Laravel-like app()
     *
     * app() вернёт контейнер.
     * app(UserService::class) создаст сервис.
     */
    function app(?string $abstract = null): mixed
    {
        if ($abstract === null) {
            return App::container();
        }

        return App::make($abstract);
    }
}


---

5. Создай demo-сервис /local/mvc_demo/Services/DemoGreetingService.php

<?php

namespace Local\MvcDemo\Services;

class DemoGreetingService
{
    public function __construct(
        private string $appName
    ) {}

    public function message(): string
    {
        return 'Привет из ServiceProvider. Приложение: ' . $this->appName;
    }
}


---

6. Создай provider /local/mvc_demo/Providers/AppServiceProvider.php

Сначала папка:

/local/mvc_demo/Providers/

Файл:

<?php

namespace Local\MvcDemo\Providers;

use Local\Mvc\Core\Config;
use Local\Mvc\Core\ServiceProvider;
use Local\MvcDemo\Services\DemoGreetingService;
use Local\MvcDemo\Services\UserService;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        /**
         * singleton — один объект на запрос.
         */
        $this->app->singleton(DemoGreetingService::class, function () {
            return new DemoGreetingService(
                (string)Config::get('app.name', 'MVC Demo')
            );
        });

        /**
         * Можно явно зарегистрировать сервис.
         * Сейчас это не обязательно, но это Laravel-like подход.
         */
        $this->app->bind(UserService::class, UserService::class);
    }

    public function boot(): void
    {
        //
    }
}


---

7. Подключи provider в /local/mvc_demo/config.php

Добавь блок:

'providers' => [
    \Local\MvcDemo\Providers\AppServiceProvider::class,
],

Полный файл должен быть примерно таким:

<?php

return [
    'app' => [
        'name' => 'MVC Demo',
        'description' => 'Тестовый проект на общем MVC-фреймворке',
    ],

    'debug' => true,

    'log' => [
        'file' => __DIR__ . '/logs/app.log',
    ],

    'providers' => [
        \Local\MvcDemo\Providers\AppServiceProvider::class,
    ],

    'roles' => [
        'resolver' => \Local\MvcDemo\Services\DemoRoleResolver::class,

        'hierarchy' => [
            'viewer' => 10,
            'editor' => 20,
            'admin' => 30,
            'owner' => 40,
        ],
    ],
];


---

8. Добавь тест в HomeController

Вверху файла добавь:

use Local\MvcDemo\Services\DemoGreetingService;

Внутрь класса добавь метод:

public function providerTest(DemoGreetingService $greeting): Response
{
    return $this->success([
        'message' => $greeting->message(),
        'service' => DemoGreetingService::class,
    ]);
}


---

9. Добавь маршрут в /local/mvc_demo/routes.php

Route::get('/provider-test', [HomeController::class, 'providerTest'])
    ->name('provider.test');


---

10. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/provider-test

Должен быть JSON примерно такой:

{
  "ok": true,
  "data": {
    "message": "Привет из ServiceProvider. Приложение: MVC Demo",
    "service": "Local\\MvcDemo\\Services\\DemoGreetingService"
  }
}


---

Что мы сделали

Теперь у нас есть Laravel-like ServiceProvider.

Было:

Container сам создаёт классы как умеет.

Стало:

Проект может явно сказать контейнеру:
"Вот этот сервис создавай вот так".

Пример:

$this->app->singleton(DemoGreetingService::class, function () {
    return new DemoGreetingService('MVC Demo');
});

Главная мысль:

Container — создаёт зависимости.
ServiceProvider — настраивает Container.

Это очень похоже на Laravel.