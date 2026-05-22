Дальше делаем App — запускатель проекта.

Зачем он нужен простыми словами:

Сейчас у каждого проекта будет похожий index.php:

$request = Request::createFromGlobals();
$router = new Router();
require_once __DIR__ . '/routes.php';
$router->dispatch($request);

Если проектов будет много:

/local/sitebuilder
/local/glab
/local/qr_opros
/local/mvc_demo

то этот код будет повторяться везде.

Поэтому делаем один общий запускатель:

App::run();


---

1. Создай файл /local/mvc/Core/App.php

<?php

namespace Local\Mvc\Core;

/**
 * App
 *
 * Это запускатель MVC-приложения.
 *
 * Простыми словами:
 * проект говорит "запусти меня",
 * а App сам создаёт Request, Router, подключает routes.php
 * и запускает нужный контроллер.
 */
class App
{
    public static function run(?string $routesFile = null): void
    {
        $projectRoot = self::projectRoot();

        if ($routesFile === null) {
            $routesFile = $projectRoot . '/routes.php';
        }

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
         * 1. Создаём объект запроса.
         */
        $request = Request::createFromGlobals();

        /**
         * 2. Создаём роутер.
         */
        $router = new Router();

        /**
         * 3. Подключаем маршруты конкретного проекта.
         *
         * Внутри routes.php будет доступна переменная $router.
         */
        require $routesFile;

        /**
         * 4. Запускаем обработку запроса.
         */
        $router->dispatch($request);
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

2. Теперь упрощаем /local/mvc_demo/index.php

Если ты уже создал /local/mvc_demo, то файл:

/local/mvc_demo/index.php

можно сделать таким:

<?php

/**
 * index.php проекта mvc_demo.
 *
 * Это входная точка конкретного проекта.
 */

define('LOCAL_MVC_PROJECT_ROOT', __DIR__);
define('LOCAL_MVC_PROJECT_URL', '/local/mvc_demo');
define('LOCAL_MVC_PROJECT_NAMESPACE', 'Local\\MvcDemo\\');

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

use Local\Mvc\Core\App;

App::run();

Вот теперь красиво.

Проект просто говорит:

Я лежу здесь.
Мой URL такой.
Мой namespace такой.
Запусти меня.

А всё остальное делает фреймворк.


---

3. Что должно остаться в /local/mvc_demo/routes.php

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;

/** @var Router $router */

$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);


---

Что мы сделали

Было:

/local/mvc_demo/index.php сам создаёт Request
/local/mvc_demo/index.php сам создаёт Router
/local/mvc_demo/index.php сам подключает routes.php
/local/mvc_demo/index.php сам запускает dispatch

Стало:

/local/mvc_demo/index.php только описывает проект
App::run() запускает всё остальное

То есть теперь каждый новый проект будет подключаться одинаково:

define('LOCAL_MVC_PROJECT_ROOT', __DIR__);
define('LOCAL_MVC_PROJECT_URL', '/local/sitebuilder');
define('LOCAL_MVC_PROJECT_NAMESPACE', 'Local\\Sitebuilder\\');

require_once $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/bootstrap.php';

use Local\Mvc\Core\App;

App::run();


---

Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Проверь:

https://bitrix24-stage.gaz.ru/local/mvc_demo/

https://bitrix24-stage.gaz.ru/local/mvc_demo/about

https://bitrix24-stage.gaz.ru/local/mvc_demo/ping

Если всё работает — следующий шаг будет очень важный: сделаем Auth-класс, чтобы фреймворк умел проверять авторизацию пользователя Битрикса.