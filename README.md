Идём дальше. Сделаем полезную вещь для разработки — страницу списка маршрутов.

Она нужна, чтобы быстро видеть:

какие URL есть
какой контроллер вызывается
какие middleware стоят

Например:

GET /admin/users              AdminController::users       auth, admin
GET /api/users/{id:\d+}       UserApiController::show      auth, admin
POST /form/send               FormController::send         csrf

Это очень помогает, когда появляется 404 и непонятно, зарегистрирован маршрут или нет.


---

1. Обновляем /local/mvc/Core/Router.php

Нам нужно добавить метод, который вернёт список маршрутов.

Открой:

/local/mvc/Core/Router.php

Внутрь класса Router добавь метод:

/**
 * Получить список всех маршрутов.
 *
 * Нужно для debug-страницы.
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

Лучше вставить его после методов get() и post().


---

2. Обновляем /local/mvc/Core/App.php

Сейчас Router создаётся внутри App::run(), и контроллеры не знают список маршрутов.

Сделаем так, чтобы App хранил текущий router.

Полностью замени файл:

/local/mvc/Core/App.php

на:

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
             * 4. Создаём Router и сохраняем его в App.
             */
            $router = new Router();
            self::$router = $router;

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

    /**
     * Получить текущий Router.
     */
    public static function router(): ?Router
    {
        return self::$router;
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

3. Создаём DebugController

Создай файл:

/local/mvc_demo/Controllers/DebugController.php

Код:

<?php

namespace Local\MvcDemo\Controllers;

use Local\Mvc\Core\App;
use Local\Mvc\Core\Config;
use Local\Mvc\Core\Controller;
use Local\Mvc\Core\Response;

class DebugController extends Controller
{
    public function routes(): Response
    {
        $router = App::router();

        return $this->render('debug/routes', [
            'title' => 'Debug: маршруты',
            'routes' => $router ? $router->routes() : [],
            'debug' => Config::debug(),
        ]);
    }
}


---

4. Создаём View

Создай папку:

/local/mvc_demo/Views/debug/

Создай файл:

/local/mvc_demo/Views/debug/routes.php

Код:

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Маршруты') ?>
    </h1>

    <p class="mvc-page-text">
        Здесь показаны маршруты текущего проекта.
    </p>

    <?php if (empty($debug)): ?>
        <div class="mvc-info" style="border-color: #fde68a; background: #fffbeb;">
            <b style="color: #92400e;">Внимание:</b>
            debug-режим выключен.
        </div>
    <?php endif; ?>

    <div class="mvc-info">
        <table style="width: 100%; border-collapse: collapse;">
            <thead>
                <tr>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Метод</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Путь</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Контроллер</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Action</th>
                    <th style="text-align: left; padding: 8px; border-bottom: 1px solid #e5e7eb;">Middleware</th>
                </tr>
            </thead>

            <tbody>
                <?php foreach (($routes ?? []) as $route): ?>
                    <tr>
                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <span class="mvc-code">
                                <?= htmlspecialcharsbx($route['method'] ?? '') ?>
                            </span>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <span class="mvc-code">
                                <?= htmlspecialcharsbx($route['path'] ?? '') ?>
                            </span>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= htmlspecialcharsbx($route['controller'] ?? '') ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?= htmlspecialcharsbx($route['action'] ?? '') ?>
                        </td>

                        <td style="padding: 8px; border-bottom: 1px solid #e5e7eb;">
                            <?php foreach (($route['middleware'] ?? []) as $middleware): ?>
                                <span class="mvc-code">
                                    <?= htmlspecialcharsbx($middleware) ?>
                                </span>
                            <?php endforeach; ?>
                        </td>
                    </tr>
                <?php endforeach; ?>
            </tbody>
        </table>
    </div>
</div>


---

5. Добавляем маршрут

Открой:

/local/mvc_demo/routes.php

Добавь use:

use Local\MvcDemo\Controllers\DebugController;

И добавь маршрут, лучше только для админа:

$router->get('/debug/routes', [DebugController::class, 'routes'], ['auth', 'admin']);

Например рядом с публичными/тестовыми маршрутами или в админской группе.


---

6. Добавь ссылку в меню

Файл:

/local/mvc_demo/Views/layouts/app.php

В меню добавь:

<a href="/local/mvc_demo/debug/routes">Routes</a>


---

7. Проверка

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/debug/routes

Должна появиться таблица всех маршрутов.


---

Что мы сделали

Теперь у нас есть debug-страница:

/local/mvc_demo/debug/routes

Она показывает:

метод
путь
контроллер
action
middleware

Это очень полезно для разработки. Когда будет 50–100 маршрутов, можно быстро проверить:

есть ли нужный маршрут
какой controller/action стоит
не забыли ли middleware
правильно ли prefix склеился

Следующим шагом можно сделать именованные маршруты, чтобы не писать ссылки руками вроде /local/mvc_demo/admin/users, а обращаться к ним по имени.