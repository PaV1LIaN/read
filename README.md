Идём дальше в сторону Laravel. Следующий кирпич — Facades.

В Laravel ты часто видишь:

Route::get(...)
Log::info(...)
Config::get(...)

То есть мы не пишем:

\Local\Mvc\Core\Logger::info(...)
\Local\Mvc\Core\Config::get(...)

А пишем короче и похоже на Laravel:

Log::info('Сообщение');
ConfigFacade::get('app.name');

У нас Route facade уже есть. Сейчас добавим ещё:

Log
ConfigFacade
AppFacade


---

1. Создай базовый facade /local/mvc/Support/Facades/Facade.php

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\App;
use RuntimeException;

/**
 * Facade
 *
 * Базовый Laravel-like facade.
 *
 * Простыми словами:
 * facade — это короткая статическая обёртка
 * над объектом из Container.
 */
abstract class Facade
{
    /**
     * Имя класса/сервиса, который нужно взять из контейнера.
     */
    abstract protected static function accessor(): string;

    public static function __callStatic(string $method, array $arguments): mixed
    {
        $accessor = static::accessor();

        if ($accessor === '') {
            throw new RuntimeException('FACADE_ACCESSOR_EMPTY: ' . static::class);
        }

        $instance = App::make($accessor);

        if (!method_exists($instance, $method)) {
            throw new RuntimeException(
                'FACADE_METHOD_NOT_FOUND: ' . static::class . '::' . $method
            );
        }

        return $instance->{$method}(...$arguments);
    }
}


---

2. Сделаем объектный логгер

Сейчас Logger у нас статический. Для facade лучше сделать обычный сервис.

Создай файл:

/local/mvc/Core/LogManager.php

<?php

namespace Local\Mvc\Core;

/**
 * LogManager
 *
 * Объектная обёртка над Logger.
 *
 * Нужна, чтобы использовать Laravel-like facade:
 * Log::info(...)
 */
class LogManager
{
    public function info(string $message, array $context = []): void
    {
        Logger::info($message, $context);
    }

    public function warning(string $message, array $context = []): void
    {
        Logger::warning($message, $context);
    }

    public function error(string $message, array $context = []): void
    {
        Logger::error($message, $context);
    }

    public function debug(string $message, array $context = []): void
    {
        Logger::debug($message, $context);
    }
}


---

3. Создай facade /local/mvc/Support/Facades/Log.php

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\LogManager;

/**
 * Log
 *
 * Laravel-like facade для логов.
 *
 * Пример:
 * Log::info('Текст');
 */
class Log extends Facade
{
    protected static function accessor(): string
    {
        return LogManager::class;
    }
}

Теперь можно будет писать:

Log::info('Что-то произошло');


---

4. Создай facade /local/mvc/Support/Facades/Config.php

<?php

namespace Local\Mvc\Support\Facades;

/**
 * Config
 *
 * Laravel-like facade для config.
 *
 * Чтобы не конфликтовать с Core\Config,
 * использовать будем так:
 *
 * use Local\Mvc\Support\Facades\Config as ConfigFacade;
 */
class Config extends Facade
{
    protected static function accessor(): string
    {
        return \Local\Mvc\Core\Config::class;
    }
}

Но у нас Core\Config пока статический класс, а facade ожидает объект. Поэтому сделаем маленький manager.


---

5. Создай /local/mvc/Core/ConfigManager.php

<?php

namespace Local\Mvc\Core;

/**
 * ConfigManager
 *
 * Объектная обёртка над Config.
 */
class ConfigManager
{
    public function get(string $key, mixed $default = null): mixed
    {
        return Config::get($key, $default);
    }

    public function debug(): bool
    {
        return Config::debug();
    }

    public function all(): array
    {
        return Config::all();
    }
}

Теперь поправь facade Config.

Полностью замени:

/local/mvc/Support/Facades/Config.php

на:

<?php

namespace Local\Mvc\Support\Facades;

use Local\Mvc\Core\ConfigManager;

/**
 * Config
 *
 * Laravel-like facade для config.
 */
class Config extends Facade
{
    protected static function accessor(): string
    {
        return ConfigManager::class;
    }
}


---

6. Создай facade /local/mvc/Support/Facades/App.php

<?php

namespace Local\Mvc\Support\Facades;

/**
 * App
 *
 * Laravel-like facade для приложения/container.
 */
class App extends Facade
{
    protected static function accessor(): string
    {
        return \Local\Mvc\Core\Container::class;
    }
}

Теперь можно будет писать:

App::make(UserService::class);


---

7. Зарегистрируй эти сервисы в Container

Открой:

/local/mvc/Core/App.php

В методе run() найди место:

$container->instance(Request::class, $request);
$container->instance(Router::class, $router);
$container->instance(Container::class, $container);

Сразу после этого добавь:

$container->singleton(\Local\Mvc\Core\LogManager::class, \Local\Mvc\Core\LogManager::class);
$container->singleton(\Local\Mvc\Core\ConfigManager::class, \Local\Mvc\Core\ConfigManager::class);

Должно быть так:

$container->instance(Request::class, $request);
$container->instance(Router::class, $router);
$container->instance(Container::class, $container);

$container->singleton(\Local\Mvc\Core\LogManager::class, \Local\Mvc\Core\LogManager::class);
$container->singleton(\Local\Mvc\Core\ConfigManager::class, \Local\Mvc\Core\ConfigManager::class);


---

8. Добавь Laravel-like helper config()

Открой:

/local/mvc/helpers.php

В конец добавь:

if (!function_exists('config')) {
    /**
     * Laravel-like config()
     *
     * config('app.name')
     */
    function config(string $key, mixed $default = null): mixed
    {
        return \Local\Mvc\Core\Config::get($key, $default);
    }
}

Теперь во views/контроллерах можно писать:

config('app.name')


---

9. Добавь тест в HomeController

Открой:

/local/mvc_demo/Controllers/HomeController.php

Сверху добавь:

use Local\Mvc\Support\Facades\Log;
use Local\Mvc\Support\Facades\Config as ConfigFacade;

Внутрь класса добавь метод:

public function facadeTest(): Response
{
    Log::info('Открыт facade-test', [
        'user_id' => \Local\Mvc\Core\Auth::id(),
    ]);

    return $this->success([
        'message' => 'Facades работают',
        'app_name_from_facade' => ConfigFacade::get('app.name'),
        'app_name_from_helper' => config('app.name'),
    ]);
}


---

10. Добавь маршрут

В /local/mvc_demo/routes.php добавь:

Route::get('/facade-test', [HomeController::class, 'facadeTest'])
    ->name('facade.test');


---

11. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

/local/mvc_demo/facade-test

Должен быть JSON:

{
  "ok": true,
  "data": {
    "message": "Facades работают",
    "app_name_from_facade": "MVC Demo",
    "app_name_from_helper": "MVC Demo"
  }
}

И в логе:

/local/mvc_demo/logs/app.log

должна появиться запись:

[INFO] Открыт facade-test ...


---

Что мы сделали

Теперь у нас есть Laravel-like facades:

Route::get(...)
Log::info(...)
ConfigFacade::get(...)

И helper:

config('app.name')

Главная мысль:

Container хранит сервисы.
Facade даёт красивый статический доступ к сервисам.

В Laravel это работает похожим образом: ты пишешь красиво Log::info(), а внутри Laravel достаёт нужный сервис из контейнера.