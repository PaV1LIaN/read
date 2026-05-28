Идём дальше. Сейчас сделаем Config — настройки проекта в одном месте.

Зачем это нужно:

Сейчас настройки разбросаны:
LOCAL_MVC_DEBUG
LOCAL_MVC_PROJECT_ROOT
LOCAL_MVC_PROJECT_URL
путь к логам
название проекта

А мы хотим, чтобы у каждого проекта был файл:

/local/mvc_demo/config.php

И там лежали настройки проекта.


---

1. Создай /local/mvc/Core/Config.php

<?php

namespace Local\Mvc\Core;

/**
 * Config
 *
 * Хранилище настроек текущего проекта.
 *
 * Простыми словами:
 * проект кладёт сюда настройки,
 * а фреймворк и контроллеры могут их читать.
 */
class Config
{
    private static array $items = [];

    /**
     * Загрузить настройки.
     */
    public static function load(array $items): void
    {
        self::$items = array_replace_recursive(self::$items, $items);
    }

    /**
     * Получить настройку.
     *
     * Пример:
     * Config::get('app.name')
     * Config::get('debug')
     * Config::get('log.file')
     */
    public static function get(string $key, mixed $default = null): mixed
    {
        $parts = explode('.', $key);

        $value = self::$items;

        foreach ($parts as $part) {
            if (!is_array($value) || !array_key_exists($part, $value)) {
                return $default;
            }

            $value = $value[$part];
        }

        return $value;
    }

    /**
     * Проверить, включён ли debug.
     */
    public static function debug(): bool
    {
        return (bool)self::get('debug', false);
    }

    /**
     * Все настройки.
     */
    public static function all(): array
    {
        return self::$items;
    }
}


---

2. Создай /local/mvc_demo/config.php

<?php

return [
    /**
     * Настройки приложения.
     */
    'app' => [
        'name' => 'MVC Demo',
        'description' => 'Тестовый проект на общем MVC-фреймворке',
    ],

    /**
     * Пока учимся — debug включён.
     * На боевом проекте ставим false.
     */
    'debug' => true,

    /**
     * Логи проекта.
     */
    'log' => [
        'file' => __DIR__ . '/logs/app.log',
    ],
];


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
    public static function run(?string $routesFile = null): void
    {
        $projectRoot = self::projectRoot();

        /**
         * 1. Загружаем config.php проекта.
         */
        self::loadConfig($projectRoot);

        if ($routesFile === null) {
            $routesFile = $projectRoot . '/routes.php';
        }

        /**
         * 2. Создаём Request.
         */
        $request = Request::createFromGlobals();

        /**
         * 3. Включаем общий обработчик ошибок.
         */
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

            /**
             * 4. Создаём Router.
             */
            $router = new Router();

            /**
             * 5. Подключаем маршруты проекта.
             */
            require $routesFile;

            /**
             * 6. Запускаем обработку запроса.
             */
            $router->dispatch($request);
        } catch (\Throwable $e) {
            ErrorHandler::renderThrowable($e);
        }
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

        /**
         * Значения по умолчанию.
         */
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

        /**
         * Значения проекта.
         */
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

4. Обнови /local/mvc/Core/Logger.php

Замени метод debug():

public static function debug(string $message, array $context = []): void
{
    if (!defined('LOCAL_MVC_DEBUG') || LOCAL_MVC_DEBUG !== true) {
        return;
    }

    self::write('DEBUG', $message, $context);
}

на:

public static function debug(string $message, array $context = []): void
{
    if (!Config::debug()) {
        return;
    }

    self::write('DEBUG', $message, $context);
}

И замени метод logFile():

private static function logFile(): string
{
    if (defined('LOCAL_MVC_LOG_FILE')) {
        return (string)LOCAL_MVC_LOG_FILE;
    }

    if (defined('LOCAL_MVC_PROJECT_ROOT')) {
        return rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/') . '/logs/app.log';
    }

    return $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/logs/app.log';
}

на:

private static function logFile(): string
{
    return (string)Config::get(
        'log.file',
        defined('LOCAL_MVC_PROJECT_ROOT')
            ? rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/') . '/logs/app.log'
            : $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/logs/app.log'
    );
}


---

5. Обнови /local/mvc/Core/ErrorHandler.php

Найди метод:

private static function debugEnabled(): bool
{
    return defined('LOCAL_MVC_DEBUG') && LOCAL_MVC_DEBUG === true;
}

Замени на:

private static function debugEnabled(): bool
{
    return Config::debug();
}

Теперь debug берётся из config.php.


---

6. Обнови /local/mvc_demo/index.php

Теперь можно убрать:

define('LOCAL_MVC_DEBUG', true);

И оставить так:

<?php

define('LOCAL_MVC_PROJECT_ROOT', __DIR__);
define('LOCAL_MVC_PROJECT_URL', '/local/mvc_demo');
define('LOCAL_MVC_PROJECT_NAMESPACE', 'Local\\MvcDemo\\');

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

use Local\Mvc\Core\App;

App::run();

Debug теперь лежит здесь:

// /local/mvc_demo/config.php
'debug' => true,


---

7. Добавим тест config

В /local/mvc_demo/Controllers/HomeController.php добавь сверху:

use Local\Mvc\Core\Config;

И добавь метод:

public function configTest(): Response
{
    return $this->success([
        'app_name' => Config::get('app.name'),
        'description' => Config::get('app.description'),
        'debug' => Config::debug(),
        'log_file' => Config::get('log.file'),
    ]);
}

В /local/mvc_demo/routes.php добавь маршрут:

$router->get('/config-test', [HomeController::class, 'configTest']);


---

8. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/config-test

Должен быть JSON примерно такой:

{
  "ok": true,
  "data": {
    "app_name": "MVC Demo",
    "description": "Тестовый проект на общем MVC-фреймворке",
    "debug": true,
    "log_file": "/srv/bx/docroot/local/mvc_demo/logs/app.log"
  }
}


---

Что мы сделали

Теперь настройки проекта лежат не россыпью по файлам, а в одном месте:

/local/mvc_demo/config.php

Фреймворк может читать:

Config::get('app.name')
Config::get('log.file')
Config::debug()

Это удобно, потому что для другого проекта будет свой config:

/local/sitebuilder/config.php
/local/glab/config.php
/local/qr_opros/config.php

А общий фреймворк /local/mvc останется один.