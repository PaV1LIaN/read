Продолжаем.

Сейчас у нас в index.php есть такое:

$method = $_SERVER['REQUEST_METHOD'] ?? 'GET';
$path = $_GET['route'] ?? '/';

Это работает, но плохо для будущего.

Почему плохо?

Потому что потом в разных файлах начнётся:

$_GET['id']
$_POST['name']
$_SERVER['REQUEST_METHOD']
$_FILES

И проект опять превратится в кашу.

Поэтому делаем специальный класс:

Request

Он будет отвечать за вопрос:

Что пришло от пользователя?

Например:

$request->get('id');
$request->post('name');
$request->method();
$request->path();
$request->isPost();


---

Шаг 1. Создай файл Request.php

Создай файл:

/local/mvc/Core/Request.php

Полный код:

<?php

namespace Local\Mvc\Core;

/**
 * Request
 *
 * Это класс для работы с запросом пользователя.
 *
 * Он аккуратно хранит:
 * - $_GET
 * - $_POST
 * - $_SERVER
 * - $_FILES
 *
 * Чтобы мы не обращались к ним напрямую по всему проекту.
 */
class Request
{
    private array $get;
    private array $post;
    private array $server;
    private array $files;

    public function __construct(array $get, array $post, array $server, array $files = [])
    {
        $this->get = $get;
        $this->post = $post;
        $this->server = $server;
        $this->files = $files;
    }

    /**
     * Создаём Request из глобальных PHP-массивов.
     *
     * То есть из:
     * $_GET
     * $_POST
     * $_SERVER
     * $_FILES
     */
    public static function createFromGlobals(): self
    {
        return new self($_GET, $_POST, $_SERVER, $_FILES);
    }

    /**
     * Получить значение из GET.
     *
     * Например:
     * /local/mvc/?route=/ping&id=5
     *
     * $request->get('id') вернёт 5
     */
    public function get(string $key, mixed $default = null): mixed
    {
        return $this->get[$key] ?? $default;
    }

    /**
     * Получить значение из POST.
     *
     * Например из формы:
     * name = "Тестовый сайт"
     *
     * $request->post('name') вернёт "Тестовый сайт"
     */
    public function post(string $key, mixed $default = null): mixed
    {
        return $this->post[$key] ?? $default;
    }

    /**
     * Получить значение сначала из POST, потом из GET.
     *
     * Удобно, когда не важно, откуда пришёл параметр.
     */
    public function input(string $key, mixed $default = null): mixed
    {
        if (array_key_exists($key, $this->post)) {
            return $this->post[$key];
        }

        if (array_key_exists($key, $this->get)) {
            return $this->get[$key];
        }

        return $default;
    }

    /**
     * Получить все GET-параметры.
     */
    public function allGet(): array
    {
        return $this->get;
    }

    /**
     * Получить все POST-параметры.
     */
    public function allPost(): array
    {
        return $this->post;
    }

    /**
     * Метод запроса.
     *
     * Обычно:
     * GET
     * POST
     */
    public function method(): string
    {
        return strtoupper((string)($this->server['REQUEST_METHOD'] ?? 'GET'));
    }

    /**
     * Проверка: это GET-запрос?
     */
    public function isGet(): bool
    {
        return $this->method() === 'GET';
    }

    /**
     * Проверка: это POST-запрос?
     */
    public function isPost(): bool
    {
        return $this->method() === 'POST';
    }

    /**
     * Получить маршрут.
     *
     * Пока мы используем такой формат:
     *
     * /local/mvc/?route=/ping
     *
     * Значит route лежит в GET.
     */
    public function path(): string
    {
        $route = (string)$this->get('route', '/');

        $route = trim($route);

        if ($route === '') {
            return '/';
        }

        return $route;
    }

    /**
     * Получить файл из $_FILES.
     *
     * Потом пригодится для загрузки файлов.
     */
    public function file(string $key): ?array
    {
        return $this->files[$key] ?? null;
    }

    /**
     * Получить значение из $_SERVER.
     */
    public function server(string $key, mixed $default = null): mixed
    {
        return $this->server[$key] ?? $default;
    }
}


---

Шаг 2. Обновляем Controller.php

Теперь контроллеры смогут получать доступ к запросу.

Замени файл:

/local/mvc/Core/Controller.php

полностью на этот:

<?php

namespace Local\Mvc\Core;

/**
 * Controller
 *
 * Базовый контроллер.
 *
 * Все остальные контроллеры наследуются от него.
 */
class Controller
{
    /**
     * Здесь будет лежать текущий запрос.
     *
     * Например:
     * $this->request->get('id')
     * $this->request->post('name')
     */
    protected Request $request;

    public function __construct(?Request $request = null)
    {
        $this->request = $request ?? Request::createFromGlobals();
    }

    /**
     * Показать HTML-страницу.
     */
    protected function render(string $view, array $params = []): void
    {
        extract($params);

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/header.php';

        require dirname(__DIR__) . '/Views/' . $view . '.php';

        require $_SERVER['DOCUMENT_ROOT'] . '/bitrix/footer.php';
    }

    /**
     * Вернуть JSON.
     */
    protected function json(array $data): void
    {
        header('Content-Type: application/json; charset=utf-8');

        echo json_encode($data, JSON_UNESCAPED_UNICODE);

        exit;
    }

    /**
     * Успешный JSON-ответ.
     */
    protected function success(array $data = []): void
    {
        $this->json([
            'ok' => true,
            'data' => $data,
        ]);
    }

    /**
     * JSON-ошибка.
     */
    protected function error(string $message, array $details = []): void
    {
        $this->json([
            'ok' => false,
            'error' => $message,
            'details' => $details,
        ]);
    }
}

Что изменилось?

Было:

class Controller
{
}

Стало:

class Controller
{
    protected Request $request;
}

Теперь любой контроллер сможет делать так:

$id = $this->request->get('id');
$name = $this->request->post('name');


---

Шаг 3. Обновляем Router.php

Теперь Router будет принимать не отдельно $method и $path, а весь объект Request.

Замени файл:

/local/mvc/Core/Router.php

полностью:

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
     * Зарегистрировать GET-маршрут.
     */
    public function get(string $path, array $handler): void
    {
        $this->add('GET', $path, $handler);
    }

    /**
     * Зарегистрировать POST-маршрут.
     */
    public function post(string $path, array $handler): void
    {
        $this->add('POST', $path, $handler);
    }

    /**
     * Добавить маршрут.
     */
    private function add(string $method, string $path, array $handler): void
    {
        $method = strtoupper($method);
        $path = $this->normalizePath($path);

        $this->routes[$method][$path] = $handler;
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

        $handler = $this->routes[$method][$path];

        $controllerClass = $handler[0] ?? null;
        $controllerMethod = $handler[1] ?? null;

        if (!$controllerClass || !class_exists($controllerClass)) {
            $this->serverError('Контроллер не найден: ' . (string)$controllerClass);
            return;
        }

        /**
         * Важно:
         * передаём Request внутрь контроллера.
         */
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

        $controller->{$controllerMethod}();
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
        http_response_code(404);

        echo '<h1>404</h1>';
        echo '<p>Маршрут не найден.</p>';
        echo '<pre>';
        echo 'Method: ' . htmlspecialchars($method) . "\n";
        echo 'Path: ' . htmlspecialchars($path) . "\n";
        echo '</pre>';

        exit;
    }

    /**
     * 500 — ошибка внутри MVC.
     */
    private function serverError(string $message): void
    {
        http_response_code(500);

        echo '<h1>500</h1>';
        echo '<p>Ошибка MVC.</p>';
        echo '<pre>' . htmlspecialchars($message) . '</pre>';

        exit;
    }
}

Главное изменение вот здесь:

public function dispatch(Request $request): void

И здесь:

$controller = new $controllerClass($request);

То есть Router теперь говорит контроллеру:

Вот тебе запрос пользователя, работай с ним.


---

Шаг 4. Обновляем index.php

Теперь входной файл станет ещё чище.

Замени файл:

/local/mvc/index.php

полностью:

<?php

/**
 * index.php
 *
 * Главная входная точка нашего MVC.
 */

require_once __DIR__ . '/bootstrap.php';

use Local\Mvc\Core\Request;
use Local\Mvc\Core\Router;

/**
 * Создаём объект запроса.
 *
 * Он внутри себя забирает:
 * $_GET
 * $_POST
 * $_SERVER
 * $_FILES
 */
$request = Request::createFromGlobals();

/**
 * Создаём Router.
 */
$router = new Router();

/**
 * Подключаем маршруты.
 */
require_once __DIR__ . '/routes.php';

/**
 * Передаём запрос в Router.
 */
$router->dispatch($request);

Теперь index.php почти идеальный.

Он делает только 4 вещи:

1. Подключил bootstrap.php
2. Создал Request
3. Создал Router
4. Запустил Router


---

Шаг 5. Обновляем HomeController.php

Добавим тест, чтобы увидеть, что Request реально работает.

Замени файл:

/local/mvc/Controllers/HomeController.php

полностью:

<?php

namespace Local\Mvc\Controllers;

use Local\Mvc\Core\Controller;

/**
 * HomeController
 *
 * Тестовый контроллер.
 */
class HomeController extends Controller
{
    public function index(): void
    {
        $name = (string)$this->request->get('name', 'Гость');

        $this->render('home/index', [
            'title' => 'MVC работает',
            'message' => 'Привет, ' . $name . '! Request успешно передан в контроллер.',
        ]);
    }

    public function ping(): void
    {
        $this->success([
            'message' => 'pong',
            'mvc' => true,
            'method' => $this->request->method(),
            'path' => $this->request->path(),
        ]);
    }
}


---

Шаг 6. Проверяем

Открой обычную страницу:

https://bitrix24-stage.gaz.ru/local/mvc/

Должно быть:

Привет, Гость! Request успешно передан в контроллер.

Теперь проверь GET-параметр:

https://bitrix24-stage.gaz.ru/local/mvc/?name=Алексей

Должно быть:

Привет, Алексей! Request успешно передан в контроллер.

Теперь проверь маршрут:

https://bitrix24-stage.gaz.ru/local/mvc/?route=/ping

Должен быть JSON примерно такой:

{
    "ok": true,
    "data": {
        "message": "pong",
        "mvc": true,
        "method": "GET",
        "path": "/ping"
    }
}


---

Что мы сейчас сделали

Раньше было так:

index.php сам лез в $_GET и $_SERVER

Теперь так:

Request забирает данные запроса
Router смотрит Request
Controller получает Request

Путь стал такой:

Браузер
  ↓
index.php
  ↓
Request
  ↓
Router
  ↓
Controller
  ↓
View / JSON


---

Самая главная мысль

Request — это пакет с информацией от пользователя.

В нём лежит:

какой адрес открыл пользователь
какой метод запроса
какие GET-параметры
какие POST-параметры
какие файлы загрузил

Теперь в будущем мы не будем писать в контроллерах так:

$_GET['id']
$_POST['name']
$_SERVER['REQUEST_METHOD']

Будем писать аккуратно:

$this->request->get('id')
$this->request->post('name')
$this->request->method()

Так код проще читать, проще проверять и проще поддерживать.

Следующим шагом сделаем Response-класс, чтобы ответы тоже были аккуратные: HTML, JSON, редирект и ошибки.