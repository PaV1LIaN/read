Дальше делаем правильный вариант:

/local/mvc        — фреймворк, инструмент
/local/mvc_demo   — первый проект, который использует этот инструмент

То есть /local/mvc больше не будет “проектом”. Он станет как мотор.
А /local/mvc_demo будет первая машинка, которая использует этот мотор.


---

Что делаем сейчас

Сейчас сделаем 3 вещи:

1. Научим /local/mvc/bootstrap.php подключать не только ядро, но и конкретный проект.
2. Переделаем Controller.php, чтобы он искал Views внутри проекта.
3. Создадим /local/mvc_demo как первый отдельный проект.


---

Шаг 1. Обновляем /local/mvc/bootstrap.php

Полностью замени:

/local/mvc/bootstrap.php

на:

<?php

/**
 * bootstrap.php
 *
 * Это запуск общего MVC-фреймворка.
 *
 * Его задача:
 * 1. Подключить Битрикс.
 * 2. Подключить автозагрузку ядра MVC.
 * 3. Если проект объявил свой namespace — подключить автозагрузку проекта.
 */

require_once $_SERVER['DOCUMENT_ROOT'] . '/bitrix/modules/main/include/prolog_before.php';

spl_autoload_register(function ($class) {
    $map = [
        'Local\\Mvc\\' => $_SERVER['DOCUMENT_ROOT'] . '/local/mvc/',
    ];

    /**
     * Если конкретный проект перед подключением bootstrap.php объявил:
     *
     * LOCAL_MVC_PROJECT_NAMESPACE
     * LOCAL_MVC_PROJECT_ROOT
     *
     * то добавляем его в автозагрузку.
     */
    if (
        defined('LOCAL_MVC_PROJECT_NAMESPACE')
        && defined('LOCAL_MVC_PROJECT_ROOT')
    ) {
        $projectNamespace = rtrim((string)LOCAL_MVC_PROJECT_NAMESPACE, '\\') . '\\';
        $projectRoot = rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/');

        $map[$projectNamespace] = $projectRoot . '/';
    }

    foreach ($map as $prefix => $baseDir) {
        if (strncmp($prefix, $class, strlen($prefix)) !== 0) {
            continue;
        }

        $relativeClass = substr($class, strlen($prefix));
        $file = rtrim($baseDir, '/') . '/' . str_replace('\\', '/', $relativeClass) . '.php';

        if (is_file($file)) {
            require_once $file;
        }

        return;
    }
});

Простыми словами

Раньше фреймворк знал только себя:

Local\Mvc\... => /local/mvc/...

Теперь проект сможет сказать:

Я проект mvc_demo.
Мои классы лежат в /local/mvc_demo.


---

Шаг 2. Обновляем /local/mvc/Core/Controller.php

Полностью замени:

/local/mvc/Core/Controller.php

на:

<?php

namespace Local\Mvc\Core;

use Bitrix\Main\Page\Asset;

/**
 * Controller
 *
 * Базовый контроллер фреймворка.
 *
 * Он общий для всех проектов.
 */
class Controller
{
    protected Request $request;

    /**
     * Layout по умолчанию.
     *
     * Будет искаться в:
     * /local/ИМЯ_ПРОЕКТА/Views/layouts/app.php
     */
    protected string $layout = 'layouts/app';

    public function __construct(?Request $request = null)
    {
        $this->request = $request ?? Request::createFromGlobals();
    }

    protected function render(string $view, array $params = [], ?string $layout = null): Response
    {
        $projectRoot = $this->projectRoot();
        $projectUrl = $this->projectUrl();

        $viewFile = $projectRoot . '/Views/' . $view . '.php';

        if (!is_file($viewFile)) {
            return Response::html(
                '<h1>500</h1><p>View не найден.</p><pre>' . htmlspecialchars($viewFile) . '</pre>',
                500
            );
        }

        $layoutName = $layout ?? $this->layout;
        $layoutFile = $projectRoot . '/Views/' . $layoutName . '.php';

        if (!is_file($layoutFile)) {
            return Response::html(
                '<h1>500</h1><p>Layout не найден.</p><pre>' . htmlspecialchars($layoutFile) . '</pre>',
                500
            );
        }

        extract($params);

        /**
         * Собираем view.
         */
        ob_start();
        require $viewFile;
        $content = ob_get_clean();

        /**
         * Подключаем CSS проекта.
         */
        $cssFile = $projectRoot . '/assets/app.css';
        $cssUrl = $projectUrl . '/assets/app.css';

        if (is_file($cssFile) && class_exists(Asset::class)) {
            Asset::getInstance()->addCss($cssUrl);
        }

        /**
         * Заголовок страницы.
         */
        global $APPLICATION;

        if (isset($APPLICATION) && is_object($APPLICATION) && isset($title)) {
            $APPLICATION->SetTitle((string)$title);
        }

        /**
         * Собираем итоговую страницу:
         * Битрикс header + layout проекта + Битрикс footer.
         */
        ob_start();

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/header.php';

        require $layoutFile;

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/footer.php';

        $html = ob_get_clean();

        return Response::html($html);
    }

    protected function json(array $data, int $status = 200): Response
    {
        return Response::json($data, $status);
    }

    protected function success(array $data = []): Response
    {
        return $this->json([
            'ok' => true,
            'data' => $data,
        ]);
    }

    protected function error(string $message, array $details = [], int $status = 400): Response
    {
        return $this->json([
            'ok' => false,
            'error' => $message,
            'details' => $details,
        ], $status);
    }

    protected function redirect(string $url): Response
    {
        return Response::redirect($url);
    }

    private function projectRoot(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_ROOT')) {
            return $_SERVER['DOCUMENT_ROOT'] . '/local/mvc';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_ROOT, '/');
    }

    private function projectUrl(): string
    {
        if (!defined('LOCAL_MVC_PROJECT_URL')) {
            return '/local/mvc';
        }

        return rtrim((string)LOCAL_MVC_PROJECT_URL, '/');
    }
}

Что изменилось

Теперь Controller больше не ищет view в:

/local/mvc/Views

Он ищет view в текущем проекте:

/local/mvc_demo/Views


---

Шаг 3. Создаём первый проект /local/mvc_demo

Создай папку:

/local/mvc_demo/

Внутри структура:

/local/mvc_demo/
  index.php
  routes.php

  Controllers/
    HomeController.php

  Views/
    layouts/
      app.php
    home/
      index.php

  assets/
    app.css


---

Шаг 4. Создай /local/mvc_demo/index.php

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

use Local\Mvc\Core\Request;
use Local\Mvc\Core\Router;

$request = Request::createFromGlobals();

$router = new Router();

require_once __DIR__ . '/routes.php';

$router->dispatch($request);


---

Шаг 5. Создай /local/mvc_demo/routes.php

<?php

use Local\Mvc\Core\Router;
use Local\MvcDemo\Controllers\HomeController;

/** @var Router $router */

$router->get('/', [HomeController::class, 'index']);

$router->get('/about', [HomeController::class, 'about']);

$router->get('/ping', [HomeController::class, 'ping']);


---

Шаг 6. Создай /local/mvc_demo/Controllers/HomeController.php

<?php

namespace Local\MvcDemo\Controllers;

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
}


---

Шаг 7. Создай /local/mvc_demo/Views/layouts/app.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-app">
    <div class="mvc-topbar">
        <div class="mvc-brand">
            <div class="mvc-brand__title">MVC Demo</div>
            <div class="mvc-brand__subtitle">Отдельный проект на общем MVC-фреймворке</div>
        </div>

        <nav class="mvc-nav">
            <a href="/local/mvc_demo/">Главная</a>
            <a href="/local/mvc_demo/about">О проекте</a>
            <a href="/local/mvc_demo/ping" target="_blank">Ping JSON</a>
        </nav>
    </div>

    <?= $content ?? '' ?>
</div>


---

Шаг 8. Создай /local/mvc_demo/Views/home/index.php

<?php

if (!defined('B_PROLOG_INCLUDED') || B_PROLOG_INCLUDED !== true) {
    die();
}

?>

<div class="mvc-card">
    <h1 class="mvc-page-title">
        <?= htmlspecialcharsbx($title ?? 'Без заголовка') ?>
    </h1>

    <p class="mvc-page-text">
        <?= htmlspecialcharsbx($message ?? '') ?>
    </p>

    <div class="mvc-info">
        <b>Что важно понять:</b>

        <ol>
            <li>
                <span class="mvc-code">/local/mvc</span>
                — это общий инструмент.
            </li>

            <li>
                <span class="mvc-code">/local/mvc_demo</span>
                — это отдельный проект.
            </li>

            <li>
                Проект подключает фреймворк через
                <span class="mvc-code">require /local/mvc/bootstrap.php</span>.
            </li>

            <li>
                Контроллер проекта наследуется от
                <span class="mvc-code">Local\Mvc\Core\Controller</span>.
            </li>
        </ol>
    </div>
</div>


---

Шаг 9. Создай /local/mvc_demo/assets/app.css

.mvc-app {
    max-width: 1180px;
    margin: 24px auto 60px;
    padding: 0 20px;
}

.mvc-topbar {
    display: flex;
    align-items: center;
    justify-content: space-between;
    gap: 20px;
    margin-bottom: 24px;
    padding: 18px 20px;
    background: #ffffff;
    border: 1px solid #e6e9ef;
    border-radius: 18px;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.06);
}

.mvc-brand {
    display: flex;
    flex-direction: column;
    gap: 4px;
}

.mvc-brand__title {
    font-size: 20px;
    font-weight: 700;
    color: #111827;
}

.mvc-brand__subtitle {
    font-size: 13px;
    color: #6b7280;
}

.mvc-nav {
    display: flex;
    align-items: center;
    gap: 8px;
    flex-wrap: wrap;
}

.mvc-nav a {
    display: inline-flex;
    align-items: center;
    min-height: 36px;
    padding: 0 14px;
    border-radius: 999px;
    background: #f3f4f6;
    color: #374151;
    text-decoration: none;
    font-size: 14px;
    transition: 0.15s ease;
}

.mvc-nav a:hover {
    background: #e5e7eb;
    color: #111827;
}

.mvc-card {
    background: #ffffff;
    border: 1px solid #e6e9ef;
    border-radius: 20px;
    padding: 28px;
    box-shadow: 0 8px 24px rgba(15, 23, 42, 0.06);
}

.mvc-page-title {
    margin: 0 0 12px;
    font-size: 28px;
    line-height: 1.2;
    color: #111827;
}

.mvc-page-text {
    margin: 0;
    font-size: 17px;
    line-height: 1.6;
    color: #4b5563;
}

.mvc-info {
    margin-top: 24px;
    padding: 18px;
    background: #f8fafc;
    border: 1px solid #e5e7eb;
    border-radius: 16px;
}

.mvc-info b {
    color: #111827;
}

.mvc-info ol {
    margin: 12px 0 0;
    padding-left: 20px;
}

.mvc-info li {
    margin: 6px 0;
    color: #4b5563;
}

.mvc-code {
    display: inline-block;
    padding: 2px 6px;
    border-radius: 6px;
    background: #eef2ff;
    color: #3730a3;
    font-family: monospace;
    font-size: 13px;
}

@media (max-width: 700px) {
    .mvc-topbar {
        align-items: flex-start;
        flex-direction: column;
    }

    .mvc-card {
        padding: 20px;
    }

    .mvc-page-title {
        font-size: 24px;
    }
}


---

Шаг 10. Добавь правило в urlrewrite.php

В файл:

/srv/bx/docroot/urlrewrite.php

в начало массива добавь:

999002 =>
array (
  'CONDITION' => '#^/local/mvc_demo/?(.*)$#',
  'RULE' => 'route=/$1',
  'ID' => '',
  'PATH' => '/local/mvc_demo/index.php',
  'SORT' => 1,
),

То есть рядом с правилом для /local/mvc.


---

Шаг 11. Проверяем

Сбрось OPcache:

opcache_reset();
echo 'OPcache reset OK';

Потом открой:

https://bitrix24-stage.gaz.ru/local/mvc_demo/

Потом:

https://bitrix24-stage.gaz.ru/local/mvc_demo/about

Потом:

https://bitrix24-stage.gaz.ru/local/mvc_demo/ping

JSON должен быть примерно такой:

{
  "ok": true,
  "data": {
    "message": "pong",
    "project": "mvc_demo",
    "framework": "local_mvc",
    "path": "/ping"
  }
}


---

Что получим после этого

У нас будет правильная архитектура:

/local/mvc
  фреймворк

/local/mvc_demo
  проект, который использует фреймворк

И дальше по такому же принципу можно делать:

/local/sitebuilder
/local/glab
/local/qr_opros

Каждый проект будет иметь свой:

index.php
routes.php
Controllers
Models
Views
assets

А общий инструмент будет один:

/local/mvc/Core